---
title: "Linux 部署 Nginx"
date: 2021-12-14T00:09:32+08:00
archives: 
    - 2021
tags:
    - Linux
image: /images/nginx.png
draft: false
---

## 安装
参考官网的[安装文档](https://nginx.org/en/linux_packages.html)。

## 静态网站
```shell
# 强制跳转HTTPS
server {
    listen 80;
    server_name example.com;
    return 301 https://$http_host$request_uri;
}

server {
    listen       443 ssl http2;
    ssl_certificate /etc/example.com.crt;
    ssl_certificate_key /etc/example.com.key;
    ssl_protocols TLSv1.3 TLSv1.2;
    
    server_name  example.com;

    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /usr/share/app/html;
        index  index.html index.htm;
    }

    #error_page  404              /404.html;
    error_page 497 https://$http_host$request_uri;
    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

## 代理网站
```shell
server{
    listen 8888 ssl http2;
    listen [::]:8888 ssl http2;
    ssl_certificate /etc/example.com.crt;
    ssl_certificate_key /etc/example.com.key;
    ssl_protocols TLSv1.3 TLSv1.2;
    
    server_name example.com;
    
    location / {
        proxy_set_header  X-Real-IP        $remote_addr;
        proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_redirect off;
        proxy_pass http://127.0.0.1:5212;
    }
    error_page 497 https://$http_host$request_uri;
}
```

## 前后端分离
```shell
server {
    listen 8934;
    listen [::]:8934;
    root /usr/share/app/dist;
    index index.html;
    
    location /api/ {
        proxy_set_header  X-Real-IP        $remote_addr;
        proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;
        proxy_redirect / /api/;
        proxy_pass http://127.0.0.1:8000/;
    }
    
    location /assets/ {
        proxy_set_header  X-Real-IP        $remote_addr;
        proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;
        proxy_pass http://127.0.0.1:8000/assets/;
    }
}
```

## 常用参数
1. `proxy_redirect`  
    修改从被代理服务器传来的应答头中的`Location`和`Refresh`字段。
    ```shell
    proxy_redirect [ default|off|redirect replacement ]
    ```
    以下两个配置等效：
    ```shell
    location /test/ {
        proxy_pass     http://s1:port/test1/;
        proxy_redirect default;
    }
    
    location /test/ {
        proxy_pass     http://s1:port/test1/;
        proxy_redirect http://s1:port/test1/ /test/;
    }
    ```
    相对跳转常用：
    ```shell
    proxy_redirect / /api/;
    ```

2. `proxy_set_header Host`  
   - `$host` 浏览器地址，没有端口
   - `$host:$proxy_port` 浏览器地址 + 代理端口
   - `$http_host` 浏览器地址 + 浏览器端口

3. `proxy_set_header X-Real-IP $remote_addr`  
   设置用户真实的地址，用于一级代理。

4. `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for`  
   记录每级代理的来源地址，以逗号分隔。真实用户地址取第一个元素。
