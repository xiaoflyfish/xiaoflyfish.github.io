---
title: TCP和IP模型
categories: 
- 网络基础
---

| OSI七层网络模型         | TCP/IP四层概念模型                   | 对应网络协议                            |
| ----------------------- | ------------------------------------ | --------------------------------------- |
| 应用层（Application）   | 应用层                               | HTTP、TFTP, FTP, NFS, WAIS、SMTP        |
| 表示层（Presentation）  | Telnet, Rlogin, SNMP, Gopher         |                                         |
| 会话层（Session）       | SMTP, DNS                            |                                         |
| 传输层（Transport）     | 传输层                               | TCP, UDP                                |
| 网络层（Network）       | 网络层                               | IP, ICMP, ARP, RARP, AKP, UUCP          |
| 数据链路层（Data Link） | 数据链路层                           | FDDI, Ethernet, Arpanet, PDN, SLIP, PPP |
| 物理层（Physical）      | IEEE 802.1A, IEEE 802.2到IEEE 802.11 |                                         |

IP、TCP、UDP、HTTP等都属于TCP/IP协议，TCP/IP泛指这些协议。

OSI模型注重通信协议必要的功能；TCP/IP更强调在计算机上实现协议应该开发哪种程序

![](https://xiaoflyfish.oss-cn-beijing.aliyuncs.com/image/20201206204022.png)