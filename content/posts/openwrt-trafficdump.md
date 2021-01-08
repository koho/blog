---
title: "OpenWrt 网关抓包"
date: 2021-01-08T11:26:10+08:00
archives: 
    - 2021
tags:
    - OpenWrt
    - 网络
draft: false
image: /images/CAAUmNO1.jpg
---

- 把 WireShark 添加到环境变量
- OpenWrt 安装 `tcpdump`

执行命令:

```shell
ssh root@openwrt tcpdump -s 0 -U -n -w - -i eth1 not port 22 | wireshark -k -i -
```
