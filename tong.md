
## Orange Pi 3B 板间 I2S 音频传输方案

### 方案概述

经过详细分析 RK3566 SoC 的所有 I2S 控制器引脚复用情况和 40-pin 排针引脚映射，**唯一可行的方案**是使用 **I2S1 控制器的 m1 复用组**。这些引脚全部暴露在 40-pin GPIO 排针上。

**代价**: 使用此方案会禁用 RK809 编解码器（耳机/扬声器音频）。HDMI 音频（I2S0）不受影响。

### 接线图

```
    Master TX (发送板)                     Slave RX (接收板)
    ┌──────────────────┐                 ┌──────────────────┐
    │ Pin 12 (SCLK)  ──┼─────────────────┼── Pin 12 (SCLK)  │
    │ GPIO3_C7         │   bit clock     │ GPIO3_C7         │
    │                  │                 │                  │
    │ Pin 35 (LRCK)  ──┼─────────────────┼── Pin 35 (LRCK)  │
    │ GPIO3_D0         │   frame sync    │ GPIO3_D0         │
    │                  │                 │                  │
    │ Pin 40 (SDO)   ──┼─────────────────┼── Pin 38 (SDI)   │
    │ GPIO3_D1         │   audio data    │ GPIO3_D2         │
    │                  │                 │                  │
    │ Pin 39 (GND)   ──┼─────────────────┼── Pin 39 (GND)   │
    └──────────────────┘                 └──────────────────┘

    可选: Pin 11 (MCLK, GPIO3_C6) 互连
```

### 已创建的文件

| 文件 | 用途 |
|------|------|
| rk356x-i2s1-ext-tx.dts | 发送板 (Master) overlay |
| rk356x-i2s1-ext-rx.dts | 接收板 (Slave) overlay |
| Makefile | 已添加两个新 overlay 编译目标 |

### 编译 & 部署

```bash
# 在 orangepi-build 目录下编译
cd /home/toxu/3b/orangepi-build
sudo ./build.sh
```

### 启用 Overlay

烧录镜像后，在每块板子的 `/boot/orangepiEnv.txt` 中添加：

**发送板 (Master TX)**:
```
overlays=i2s1-ext-tx
```

**接收板 (Slave RX)**:
```
overlays=i2s1-ext-rx
```

### 使用方法

**发送板 - 播放音频**:
```bash
# 查看声卡
aplay -l
# 播放 WAV 文件 (48kHz, 16-bit, 立体声)
aplay -D hw:0,0 -f S16_LE -r 48000 -c 2 test.wav
# 或用 sox 生成测试正弦波
sox -n -t wav -r 48000 -c 2 - synth 5 sine 1000 | aplay -D hw:0,0
```

**接收板 - 录制音频**:
```bash
# 查看声卡
arecord -l
# 录制为 WAV (参数必须与发送端匹配)
arecord -D hw:0,0 -f S16_LE -r 48000 -c 2 -d 10 recorded.wav
```

### 为什么其他 I2S 不可用

| 控制器 | 引脚组 | 状态 |
|--------|--------|------|
| I2S0 | 专用 HDMI | 已用于 HDMI 音频 |
| I2S1 m0 | GPIO1_A2-B3 | 已用于 RK809 编解码器，引脚不在排针上 |
| **I2S1 m1** | **GPIO3_C6/C7/D0-D2** | **全部在排针上，无冲突** |
| I2S2 m0 | GPIO2_B7/C0-C5 | 与蓝牙冲突 |
| I2S2 m1 | GPIO4_B6/B7/C1 | SCLK/MCLK 不在排针上 |
| I2S3 m0 | GPIO3_A2-A6 | 不在 40-pin 排针上 |
| I2S3 m1 | GPIO4_C2-C6 | LRCK 与网口 PHY reset 冲突 |





这两个 overlay 完全复用了内核中已有的驱动模块，当前 kernel config 中全部已启用：

| 功能 | 内核驱动 | Config 选项 | 当前状态 |
|------|----------|-------------|----------|
| I2S 控制器 | rockchip_i2s_tdm.c | `CONFIG_SND_SOC_ROCKCHIP_I2S_TDM` | `=m` |
| TX 虚拟编解码 | `spdif_transmitter.c` | `CONFIG_SND_SOC_SPDIF` | `=m` |
| RX 虚拟编解码 | `spdif_receiver.c` | `CONFIG_SND_SOC_SPDIF` | `=m` |
| 声卡绑定 | `simple-card.c` | `CONFIG_SND_SIMPLE_CARD` | `=m` |
| 声卡工具 | `simple-card-utils.c` | `CONFIG_SND_SIMPLE_CARD_UTILS` | `=m` |

DTS overlay 的作用就是通过设备树描述告诉这些已有驱动：用哪个 I2S 控制器、走哪组引脚复用、搭配哪个 codec。内核加载 overlay 后会自动 probe 对应的驱动模块，注册 ALSA 声卡设备。

直接编译部署即可。
