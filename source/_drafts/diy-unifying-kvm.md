---
title: 一个键盘三台电脑：用 nRF52 自制 Unifying 无线 KVM
date: 2026-06-06 17:39:30
tags:
  - nRF52
  - Rust
  - 嵌入式
  - DIY
  - KVM
---

## 背景

我有一把 ikbc c87 的有线键盘，服役好多年了一直没坏，没动力换把新的。
但是我经常要在不同的设备间切换：PC / RPI / MBP。每次都将线插来插去的，很麻烦。

尝试弄了一个有线的 usb 切换器：
![](file-20260606175842891.png)

以为能彻底解决问题，结果发现桌面上多了很多线：
* 每个设备 1 条线 x 3
* 供电线 x1
* 切换按钮线 x1

这也太不优雅了吧。

## 无线键盘协议

无线键盘主要有两种传输形式：蓝牙 和 2.4G 私有协议。

蓝牙的问题：容易被干扰、多设备切换慢（经常要等好几秒才连上）、需要目标电脑有蓝牙。

2.4G 私有协议各家有自己的方案。我的鼠标是罗技的，用的是优联（Unifying）接收器。优联接收器最多支持 6 个设备，插到电脑上就是即插即用，不需要装驱动。

那能不能**模拟一个优联键盘**，把有线键盘变成无线的呢？

## 优联协议

Logitech Unifying 是罗技在 2009 年推出的 2.4GHz 无线协议。一个接收器配对最多 6 个设备（键盘、鼠标、触控板），设备端用的是 Nordic 的 nRF24 系列芯片。

看到社区已经有成熟的研究和方案实现：
[decrazyo/unifying: FOSS re-implementation of the Logitech Unifying protocol](https://github.com/decrazyo/unifying)
在Arduino + nRF24L01 上跑通了完整的配对和加密键击

我基于它用 Rust 的embedding生态重写了协议层（[rust-unifying](https://github.com/ZHLHZHU/rust-unifying)），架构上做了硬件无关的抽象：

```mermaid
graph TD
    A[UnifyingDevice 协议状态机] --> B[trait UnifyingRadio]
    A --> C[trait Clock]
    A --> D[trait AesEncryptor]
    B --> E[nRF24L01 / SPI]
    B --> F[nRF52840 / 片上射频]
    C --> G[std::time / embassy-time]
    D --> H[aes + ctr crate]
```

协议逻辑（配对握手、AES 密钥派生、加密键击、保活、HID++ 应答）全部封在 `UnifyingDevice` 里，只通过三个 trait 触及硬件。这为后面的迁移省了大量工作——换硬件只需要重新实现一个 trait。

这套方案我在 树莓派 GPIO + nRF24L01上面跑通了，实现了配对和键盘输入，完整模拟了键盘的行为。
但 nRF24L01 跟 树莓派 SPI 通讯用的是杜邦线接起来的，不太稳定且不优雅。
![](c31a758326f976695c6360e7cffebe18.jpg)

## 迁移到 nRF52840 片上射频

nRF52840 的片上 2.4GHz RADIO 可以配置成与 nRF24L01+ 完全兼容的 ESB 模式—— 一块 U 盘大小的 nRF52840  直接插在 Pi 的 USB 口，整洁多了。
![](file-20260606190503999.png)
### 手写 ESB 驱动

现有的 `esb` crate（2020 年）跟 embassy 的 PAC 版本冲突，没法直接用。
所以我对着 nRF52840 的 PAC 手写了一个最小的 ESB PTX（主发射端）驱动。主要配置：

- 2Mbit Nordic 私有模式
- 6-bit 长度字段 + 3-bit S1（PID + no_ack）
- CRC-16（多项式 0x11021，初值 0xFFFF）
- 大端、关闭白化

最难的点是确定地址编码。nRF52 的 BASE/PREFIX 拆分方式、字节序、位序都跟 nRF24L01+ 不一样，有好几种"看似合理"的组合。

经过穷举验证确定是 `RADIOTEST ENC=1 ACKS=2`

### 保活

优联连接靠心跳包维持跳频同步。真实键盘每 ~20ms 发一个保活包，接收器就判定设备掉线。
刚开始 AI 没有做保活，断了就重连，结果越改越差，多轮重试还互相打断接收器正在重建的同步。
最后用 embassy 的 `select(read_packet, Timer::after(8ms))`，实现了保活

### 端到端验证

将 nrf 52 和 优联接收器都插入树莓派，让 ai 在上面自己一直调试，跑了一个晚上跑通了：
原生射频 → 配对 → AES 密钥派生 → 加密键击 → 接收器解密 → HID 注入。

## 上位机：KVM 转发器

固件通过 USB-CDC 串口暴露命令接口。上位机是一个 Python 脚本，用 `evdev` 抓取物理键盘，把每个按键状态变化转成 `UKEYDOWN` 命令转发给 nRF52。

### 架构

```mermaid
graph LR
    KB[物理键盘] -->|evdev grab| KVM[kvm.py]
    KVM -->|CDC 串口| NRF[nRF52840]
    NRF -->|ESB 2.4GHz| RX1[接收器 PC]
    NRF -->|ESB 2.4GHz| RX2[接收器 Mac]
    KVM -->|uinput| LOCAL[Pi 本机]
    SL[Scroll Lock] -.->|切换| KVM
```

### 多设备切换

Scroll Lock 这个案件和对应的指示灯应该是键盘上最摸鱼的部分了，将他们利用起来作为键盘设备切换和指示：
	按 **Scroll Lock** 循环切换目标（PC → 本机 → Mac → PC ...）。切换时 Scroll Lock 灯闪烁指示当前目标（1闪=PC，2闪=本机，3闪=Mac）。

固件支持 4 个独立的配对 slot，可以在 4 个设备中循环切换
![](file-20260606194343686.png)

## 持久化与防重放

优联用 AES-128-CTR 加密键击，每帧的 counter 单调递增。接收器记住见过的最大 counter，拒绝任何不大于它的帧（防重放）。

真实罗技键盘用电池供电，RAM 常驻，将 counter 放到 ram 中就可以了。但咱们方案是 USB 供电，会有断电的情况，所以需要往 flash 持久化 counter。

三重保护：

1. **启动前跳 +512**：每次从 flash 加载 counter 后立即加 512。即使断电前有未存的增量，跳 512 也能解决一大部分问题
2. **每 128 帧存一次**：正常打字约每 10-30 秒触发一次写入，flash 磨损可以接受
3. **POFCON 掉电紧急存**：VDD 跌破 2.7V 时触发中断，具体没测电容能顶多久，应该几百微秒的窗口是有的，够写 32 字节记录。

为减少 flash 磨损，存储用追加式日志：一页 4K 分成 128 个槽位，每次追加一条，写满才擦页。

## 最终效果

```
┌─────────────────────────────────┐
│       物理键盘 (ikbc c87)        │
└──────────────┬──────────────────┘
               │ USB
┌──────────────▼──────────────────┐
│     树莓派 + nRF52840 dongle     │
│     (kvm.py + systemd 常驻)      │
└──────────────┬──────────────────┘
               │ 2.4GHz ESB
    ┌──────────┼──────────┐
    ▼          ▼          ▼
  [PC]      [Pi本机]    [MacBook]
```

- **切换延迟**：按 Scroll Lock → LED 闪 → 打字即达（< 300ms）
- **输入延迟**：端到端 3-8ms，人完全无感
- **回报率**：~150Hz，日常打字绰绰有余
- **CPU 占用**：0%（epoll 阻塞，不轮询）
- **成本**：Pi Zero (~$5) + nRF52840 dongle (~$8) + 优联接收器 (~$10×2) ≈ **$33**

## 复现

硬件清单：
- 任意Linux 机器，带 usb 就行
- nRF52840 开发板/dongle（支持 USB CDC）
- Logitech Unifying 接收器，每台远程电脑一个

```bash
# 克隆项目
git clone --recursive https://github.com/ZHLHZHU/nrf52-unifying.git
git clone https://github.com/ZHLHZHU/unifying-kvm.git

# 编译固件
cd nrf52-unifying
cargo build -p nrf-demo-app --release

# 首次 SWD 烧录后,后续走 OTA
python3 tools/cdc_dfu.py --port /dev/ttyACM0 --image app.bin

# 配对接收器(接收器侧先进入配对模式)
python3 tools/unifying.py raw "UPAIR 00"   # PC
python3 tools/unifying.py raw "UPAIR 01"   # Mac

# 启动 KVM
sudo python3 unifying-kvm/kvm.py --keyboard VID:PID --targets "pc:0,local:_,mac:1"
```


## 写在最后

整个项目的代码量不大（固件 ~800 行 Rust，KVM ~400 行 Python），但涉及的层次很多：USB CDC、embedded Rust、射频协议、AES 加密、flash 持久化、Linux evdev/uinput、systemd。最有意思的部分是射频层——把一个"不确定正确编码"的问题塞进一份固件让硬件投票，以及从"事后自愈"走错路到"源头保活"的认知转折。

三个仓库都在 GitHub 上,欢迎 star / issue:

- [nrf52-unifying](https://github.com/ZHLHZHU/nrf52-unifying) — nRF52840 固件
- [rust-unifying](https://github.com/ZHLHZHU/rust-unifying) — 优联协议库
- [unifying-kvm](https://github.com/ZHLHZHU/unifying-kvm) — 键盘 KVM 转发器

---

> 本项目仅供个人学习和自用设备互操作。Logitech、Unifying 是罗技的注册商标。协议细节参考自公开的安全研究文献。
