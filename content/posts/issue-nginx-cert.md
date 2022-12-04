---
title: "为 NGINX 签发 HTTPS 证书"
date: 2022-12-05T00:19:04+08:00
archives: 
    - 2022
tags:
    - Linux
image: /images/placeholder.png
leading: false
draft: false
---
 
目前有很多 CA 可以免费签发 3 个月的 HTTPS 证书，如 Let's Encrypt，ZeroSSL 等。推荐使用 [acme.sh](https://github.com/acmesh-official/acme.sh) 脚本来管理证书，可以简化签发流程，证书到期可以自动续签。

## 安装

```shell
curl https://get.acme.sh | sh
```

## Let's Encrypt

域名验证方式有很多种，以阿里云的 DNS 认证为例：

```shell
export Ali_Key=""
export Ali_Secret=""
```

签发指定域名证书：

```shell
/root/.acme.sh/acme.sh --server letsencrypt --issue --dns dns_ali -d a.example.com
```

成功后会输出证书保存目录，而且脚本在证书到期前会自动续签证书。

安装证书到指定目录：

```shell
/root/.acme.sh/acme.sh --install-cert -d a.example.com --fullchain-file /etc/a.example.com.cer --key-file /etc/a.example.com.key --reloadcmd "systemctl restart nginx"
```

当证书续签后，上面的安装命令也会再次执行。

## Nginx 配置

```
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    ssl_certificate /etc/a.example.com.cer;
    ssl_certificate_key /etc/a.example.com.key;
    ssl_protocols TLSv1.3 TLSv1.2;

    server_name a.example.com;
    ...
}
```
