---
title: 如何优雅地开机
date: 2021-12-20 08:26:00
categories: [折腾]
tags: [WOL, ESP8266, MQTT, DIY]
---
当你不在家，却想要给家里的电脑开机，你有以下几个方案：
### 让家里人帮忙
优点：可靠，有即时的反馈告诉你电脑状态
缺点：你得确保你家里有人，独居的朋友忽略这条
### 智能插座+上电自启
在bios里面设置好上电启动，当智能插座打开的时候电脑就自动开机了。不需要靠别人，自己随时能控制
不足之处是无法直接唤醒系统S3睡眠。另外还得额外用个智能插座来实现
### 开机棒
开机棒就是用一个MCU去控制你电脑的开机跳线，给MCU发送指令后导通主板跳线，机器开机。相当于帮你按了开机键
![Wakener_taobao](Wakener_taobao-f7913ed00d404a2f80ca56a0aff59684.png)
缺点就是对电脑有侵入修改，如果你有多台电脑就要准备多个开机棒
### 局域网唤醒（Wake-on-LAN/WOL）
优点：可以自己控制，可以直接唤醒S3状态下的电脑，设备数量不限，只要在同一个广播域，都能唤醒
缺点：得在目标电脑广播域中有一个可以发唤醒包的设备，可以是路由器，旧手机，树莓派等等。。。

综合起来看，WOL的方案比较合适。因此某天跟某位不愿透露姓名的[SwingFrog](http://www.swingfrog.com)聊天的时候突发奇想，不如做一个小模块来实现WOL


<!--more-->


## WOL原理
如果在BIOS中打开了WOL功能，电脑处于关闭或者休眠状态时，有线网卡仍然保持供电，网卡会监听网络的广播信息，如果发现广播的数据包为Magic Packet，并且指定的MAC地址为本机时，网卡将通知电脑开始启动。
### Magic Packet介绍
WOL唤醒数据包称为魔法数据包（Magic Packet），是一个广播帧，一般使用无状态的传输协议（如UDP），通过7或者9端口进行广播
### Magic Packet的格式
魔法数据包的格式很简单：

|Synchronization Stream |Target MAC |Password (optional) |
|:-:|:-:| :-: |
|6|96|0,4 or 6|

开头是6个字节的0xFF，
紧接着是需要唤醒主机MAC地址，重复写16次，96个字节，
最后的密码是可选项，如果配置了就只能是4或者6个字节。如果为4字节密码，将会解析为IP地址，如果为6字节密码，将会解析为MAC地址。

## 设计
设计一个唤醒器，需要实现三个功能
 - 连上网络
 - 接受外部指令
 - 在内网中广播魔法数据包
对应的，构思出整体结构：
![WakenerArch](WakenerArch-91d8a1420a7c42cfa2f0e353ec3766b8.png)
使用烂大街又好用又实惠的ESP8266作为主控，连上WIFI，订阅MQTT Topic，收到指令后通过wifi向局域网广播魔法唤醒包。
### PCB
简单做一个USB供电的驱动电路，再加一个拨码开关，用于切换配置模式和正常工作模式。
BOM表如下

| Comment     | Designator | Footprint                               | Quantity |
|-------------|------------|-----------------------------------------|----------|
| SW_DIP_x01  | SW1,       | SW_SMT_1P                               | 1 |
| 10uF        | C1,C2,     | C_0603_1608Metric                       | 2 |
| 10K         | R1,R2,R3,  | R_0603_1608Metric                       | 3 |
| RT9013-33GB | U1,        | SOT-23-5_HandSoldering                  | 1 |
| USB_A       | J1,        | USB_A_CNCTech_1001-011-01101_Horizontal | 1 |
| ESP-12F     | U2,        | ESP-12E                                 | 1 |

Schematic:
<img src="WakenerSchematic-917d054b2f4c414abc2426c3eb190a5e.png" width="600" alt="">


成品做出来是这样的：
<img src="WakenerPhysical-30844f584c894543b8f7d71a845a811e.jpg" width="450" alt="">

### 软件部分
给唤醒器设计了两种工作模式，当上电时候MCU检测拨码开关拨到ON，将会进入配置模式。
配置模式会开启AP，使用其他设备连接上去可以进行配置

上电时拨码开关关闭时，将进入工作模式，MCU会连接WiFi、链接MQTT服务器、订阅MQTT topic
唤醒指令发下后，唤醒器将会在局域网中广播
<img src="WakenerPacket-9bfb1b23e31b4c9e8bbe2a31ba529a71.png" width="600" alt="">

## 总结
小小东西鸽了好久，还是太菜了。另外可以加个mos管，引出两个针脚，就可以实现开机棒的功能。
本项目的PCB和源码都托管在GitHub上：
https://github.com/ZHLHZHU/ESPWakener
