---
title: "Linux 部署 WireGuard"
date: 2021-09-03T14:33:27+08:00
archives: 
    - 2021
tags:
    - Linux
    - 网络
image: /images/wireguard-ubuntu.jpg
draft: false
---

## 安装
### CentOS
```shell
sudo yum install epel-release elrepo-release
sudo yum install yum-plugin-elrepo
sudo yum install kmod-wireguard wireguard-tools
```

### Ubuntu
```shell
sudo apt install wireguard
```

若内核太老不想升级，可以使用 [wireguard-go](https://git.zx2c4.com/wireguard-go/about/) 代替。

## 配置
1. 创建 Wireguard 目录：
```shell
mkdir /etc/wireguard/
cd /etc/wireguard/
```

2. 生成密钥对
```shell
wg genkey | tee privatekey | wg pubkey > publickey
```

3. 编辑配置文件
```shell
vi wg0.conf
```
内容模板：
```ini
[Interface]
Address = 10.100.100.3/32
PrivateKey = KEY
ListenPort = 21841
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = PUB
Endpoint = IP
AllowedIPs = 10.100.100.3/24
PersistentKeepalive = 25
```
若防火墙已经手动配置好，则不需要`PostUp`和`PostDown`两句。关于`AllowedIPs`的简单解析：
> It acts as a routing table when sending, and an ACL when receiving. When a peer tries to send a packet to an IP, it will check AllowedIPs, and if the IP appears in the list, it will send it through the WireGuard interface. When it receives a packet over the interface, it will check AllowedIPs again, and if the packet’s source address is not in the list, it will be dropped.

4. 开启 IP 转发
```shell
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf
sysctl -p
```

6. 启动服务
```shell
sudo systemctl enable wg-quick@wg0.service
sudo systemctl start wg-quick@wg0.service
sudo systemctl status wg-quick@wg0.service
```

7. 添加 DNS 脚本

如果连接时使用了域名，WireGuard 连接时会自动解析成 IP。但当域名的 IP 发生变动时，WireGuard 并不会重新解析域名。所以对于使用 DDNS 的用户，必须添加一个 DNS 解析脚本：
```shell
wget https://raw.githubusercontent.com/WireGuard/wireguard-tools/master/contrib/reresolve-dns/reresolve-dns.sh
crontab -e
```
计划任务添加一行：
```
*/1 * * * * sh /etc/wireguard/reresolve-dns.sh wg0
```
注意`reresolve-dns.sh`脚本里用到的`$EPOCHSECONDS`这个变量，如果你的系统不支持，需要替换成`$(date +%s)`。

**参考链接**：

[1] [How to easily configure WireGuard](https://www.stavros.io/posts/how-to-configure-wireguard/)
