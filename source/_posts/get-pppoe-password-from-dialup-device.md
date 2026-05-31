---
title: 从拨号设备获取PPPoE密码
date: 2020-04-03 23:45:00
categories: [网络]
tags: [PPPoE, 宽带, 路由器, 抓包]
---
现在某些路由器有从旧路由器自动学习宽带账户密码功能，
![哇为路由截图.jpeg](RoutePPPoeExample-598850f7aa1547d38a584cc7e57c9788.jpeg)

目前大部分宽带拨号使用的PPP认证类型还是基于明文的PAP验证，只要伪装成PPPoE服务器，完成握手过程，就能获得路由器上的宽带账户与密码。
<!--more-->
## PPPoE
PPPoE是在以太网中实现PPP的数据链路层协议，数据帧结构如图
![PPPoE帧.jpg](PPPoEFrame-2380be13aea34d3fb24e9e736614cab7.jpg)
 - Len/Type字段标识PPPoE的连接阶段
 - Ver字段恒为0x01
 - Type字段恒为0x01
 - Code字段表明帧的类型
 - SessionID字段标识当前会话，由服务端在PADS阶段生成
 - Length字段表明接下来Payload的长度
 - Payload在发现阶段携带额外数据，比如服务器名；在PPP会话携带PPP内容

## PPPoE发现阶段
### PADI
当路由器上电后，路由器会以一定的间隔广播PADI包
code为0x09,Session ID为0x00
![PADI.png](PADI-034dccddeafc4cc89eac1e5d976510d6.png)

### PADO
服务器接受到广播包后，获得路由器MAC地址，并向路由器发送PADO包
code为0x07,Session ID为0x00
![PADO.png](PADO-0c24ea925afd4d18acc8f099a1639f9a.png)
### PADR
路由器接受到来自服务器的PADO后，获得服务器MAC地址，并向服务器发送PADR包
code为0x19,Session ID为0x00
![PADR.png](PADR-a7e505aa97c84a919d4ad089b5e91ac8.png)

### PADS
服务器收到PADR后，随机生成一个Session ID,放入PADS包发给路由器
code为0x65,Session ID为随机生成数字
![PADS.png](PADS-1448f115ffef4df1a6771e09e56cc04b.png)

**到此PPPoE发现阶段结束**
## PPPoE会话阶段
## LCP Configuration Request
告知对方最大接收单元（MRU）,认证协议，一个标识连接的MagicNumber
![LCPCR.png](LCPConfReq-abd9600f5751406eb4f170711b10c0f0.png)

## LCP Configuration Ack
表示支持对方的认证协议，返回的Options应该与对方发过来的一致
![LCPACK.png](LCPConfAck-018ea2a731d84456b46c0e3d417ecb6e.png)

## PAP Authentication Protocol
### Authentication-Request
此报文将会明文携带拨号账户与密码，在Peer-ID和Password字段中提取即可
![PAPRequest.png](AuthReq-249aa1155c2a41e3bd9f2d860f1f7d95.png)

## 结束会话
由于我们已经拿到账户密码，后续会话保持就不需要了
我们直接发送PAP Authentication-NAK告知对方密码错误~~（得了便宜还卖乖~~
![Authentication-NAK.png](AuthNAK-6e9b95cf7b1b42a9aca215d004d41c52.png)
然后发送LCP Termination Request结束LCP会话
![Termination-Request.png](LCPEnd-1c9c804f17a943b99cfa235515db39e0.png)

##相关工具
我用Golang写了一个工具实现以上过程以获取账户密码，请合法使用:
https://github.com/ZHLHZHU/PPPoE_Account
![PPPoE_Account](https://github.com/ZHLHZHU/PPPoE_Account/workflows/PPPoE_Account/badge.svg)

## 参考

 - [PPPoE协议-轮子学长-CSDN](https://blog.csdn.net/windeal3203/article/details/51066430)
 - [RFC2516](https://tools.ietf.org/pdf/rfc2516.pdf)
