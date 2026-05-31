---
title: 用x86工控板配置一个stratum-1 ntp服务器
date: 2019-06-11 00:44:00
categories: [网络]
tags: [NTP, Stratum-1, GPS, x86]
---
## 前言
　　国内ntp服务器的资源严重不足，在pool.ntp.org池中的每台服务器负载都很重，之前尝试过把一台1C1G1M服务器加入ntp池中，大量包直接把服务器给搞宕，然后分数直掉，被踢出ntp池，大家如果有空闲的服务器资源可以贡献出来分担一下：[ntp pool](https://www.ntppool.org/zh/)
## 那我们自己做个ntp服务器吧
　　ntp服务器采用分层结构，stratum-0是参考时钟源，参考时钟常用的是GPS信号；因为我们无法直接通过GPS信号在网络中同步时间，所以就需要stratum-1服务器作为转接，stratum-1是直接与参考时钟源相连接的。
　　在网上很多教程都是用树莓派+GPS模块来搭建stratum-1服务器的，之前尝试过，但是由于树莓派处理能力相对较低、易损坏的 SD卡、缺乏电池备份，因此我把目光投向了x86平台，本次教程使用的是一张INTEL ATOM N570工控主板。


<!--more-->


## 材料准备
　　首先我们需要一个带PPS输出的GPS模块:
![GPSMOD](GPSMOD.jpg)
　　利用GPS模块配套的软件把PPS脉冲改为周期为一秒，高电平100ms，把波特率改为115200，顺便只保留串口输出GPS的NMEA数据,因为北斗和伽利略的数据对我们无用。
　　然后通过usb线将GPS模块连接到工控板上，另外用一根杜邦线把GPS模块的PPS输出连接到工控板RS232接口的第一脚（数据载波检测DCD）上。
## 配置GPS
　　当我们连接正确之后,执行`ls /dev/tty*`就会看到一个新的设备，我这里显示的是`/dev/ttyACM0`。然后安装minicom来查看GPS模块输出是否正常
```bash
sudo apt install minicom
sudo monicom -s
```
选择`*Serial port setup`把串口设备改成`/dev/ttyACM0`，然后把波特率改成115200，回车保存，选择exit就可以看到有数据输出了（`/dev/ttyACM0`根据每个人电脑的情况自行修改）
![GPSSerialPort](GPSSerialPort-034c9e11c8544088804a913f9c5f4013.png)
`ctrl+A Z X`退出minicom（比VIM退出还要丧心病狂）

## 配置PPS
　　GPS的PPS引脚会在产生频率为1HZ的稳定方波，可以提供给工控板一个精准的走时。x86工控板不同于树莓派，没有引出GPIO口，因此我们的PPS信号要通过RS232的第一脚进行输入，并且通过`ldattach`把pps功能附加到RS232接口上。
  
```bash
sudo apt install pps-tools
modprobe pps_ldisc
lsmod | grep pps_ldisc
sudo ldattach pps /dev/ttyS0 #后面的tty设备取决于pps信号接在哪个com口上
sudo ppstest /dev/pps0
```
如果输出如图，pps就配置完成了：
![PPSDevice](PPSDevice-953ed75b626d46a1a71e373b3a318666.png)

![PPSOutput](PPSOutput-481b9e7873204c8491f9eea3b2f02360.png)

## 获取ntpd
Ubuntu18.04 Server没有自带ntp套件，因此需要我们自己安装：
从官网下载最新版ntp:http://www.ntp.org/downloads.html
``` bash
解压…
./configure
make -j4
sudo make install
```
## 配置
### 首先完成以下项目：
* GPS模块能正常搜星定位
* GPS模块设置PPS占空比为10%（看对应的GPS模块手册）
* GPS设置为动态（车载）模式（看GPS模块手册）
### 配置ntpd
在/etc/下创建ntp.conf并写入如下内容：
```bash
# gps: /dev/gps0->/dev/ttyACM0
server 127.127.20.0 mode 88 minpoll 4 iburst prefer true
fudge 127.127.20.0 flag1 1 flag2 0 flag3 0 flag4 0 time1 0.1 refid GPS
# pps: /dev/pps0
server 127.127.22.0 minpoll 4 maxpoll 4 iburst true
fudge 127.127.22.0 flag2 0 flag3 0 flag4 1 time1 0.1 refid PPS
```
ntpd配置server中的127.127.X.X并不是真实的ip地址(127开头的根本不是一个网络地址)，而是参考时钟源
127.127.22.0表示把/dev/pps0作为PPS参考源
127.127.20.0表示从/dev/gps0作为GPS参考源
**注意！我们在之前配置的时候串行输出口是/dev/ttyACM0，但是ntpd只会把/dev/gps\*中作为参考源**
**因此！我们需要做一个软连接：**
``` bash
sudo ln -s /dev/ttyACM0 /dev/gps0 #根据具体情况自行修改
```
127.127.20.0 mode 88中的mode用于配置波特率和选择NMEA字段
更多的配置信息可以参考ntpd官方文档：[GPS](https://www.eecis.udel.edu/~mills/ntp/html/drivers/driver20.html)  / [PPS](https://www.eecis.udel.edu/~mills/ntp/html/drivers/driver22.html)
## 测试
配置完成后我们来做一下测试：
```bash
sudo ntpd -c /etc/ntp.conf #启动ntpd
ntpq -crv -p
```
ntpq输出结果如图：
![NTPData](NTPData-a52965cf146c4235a7bdfcd94d916b32.png)
ntpq 各个参数详解：[网络时间的那些事及 ntpq 详解](https://linux.cn/article-4664-1.html)
拿windows测试一下：
![NTPWindows](NTPWindows-bfb3fcf5449f460d9fdf85cc34a699c6.png)


**至此就完成ntpd的基础配置啦**
