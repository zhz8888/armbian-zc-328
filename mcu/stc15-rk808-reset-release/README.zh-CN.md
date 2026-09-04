# STC15 RK808 reset-release 辅助固件

[English](README.md) | **简体中文**

本目录包含一个可用 SDCC 构建的 STC15/8051 固件，用于 ZC-328 广告屏板卡上的 STC15 MCU（该 MCU 连接在 RK808 PMIC 的 `VPPOTP` / reset 相关网络上）。

## 目录

- [为什么需要这个固件](#为什么需要这个固件)
- [工作原理](#工作原理)
- [仓库结构](#仓库结构)
- [构建](#构建)
- [刷写](#刷写)
- [注意事项](#注意事项)

## 为什么需要这个固件

原始 STC 固件看起来会把 `P1.3` / `RK808_VPP_OTP` 拉高，然后进入 power-down。而在 RK808 参考复位电路中，`VPPOTP` 同时被用作高电平复位输入：

```text
VCCRTC -- 10k -- RESET button -- VPPOTP
```

因此把这个信号拉高，就相当于一直按着 RK808 的复位键，会导致 RK3288 主板无法正常运行。

目前无法读出该 MCU 的原始固件，而它实现的看门狗在大约 40 秒内就会重启一次 RK808 PMIC。因此要正常使用这块板子，**必须**重新刷写这个辅助固件。

## 工作原理

替代固件不再拉高该信号，而是释放它：

- 在切换引脚模式前，先释放 `P1.3` 端口锁存器（`P1_3 = 1`）。
- 通过 `P1M1`/`P1M0` 把 `P1.3` 配置为高阻输入（STC15 上 `10` = 高阻输入）。
- 然后让 STC15 进入 power-down 模式。

这样尽量模拟复位按钮未按下的状态，既不会把 `VPPOTP` 拉高，也不会与经过 10k 电阻的物理复位按键路径冲突。具体的位操作请参见 `main.c`。

## 仓库结构

- `main.c` - 最小 reset-release 固件源码
- `stc15.h` - SDCC 所需的 STC15 SFR 定义（`P1M1`/`P1M0`，位于 `0x91`/`0x92`）
- `Makefile` - SDCC/packihx 构建规则，以及 `flash` 烧写目标
- `firmware.hex` - 预构建 Intel HEX（由本目录源码构建）

预构建 `firmware.hex` 的 sha256：

```text
5f8dc69edb56a1263b67e879ccf881bb6596b2502c85cd48847909888f07ae5b
```

## 构建

在 Debian/Ubuntu 上：

```bash
sudo apt install sdcc
make
```

`make clean` 会删除所有 SDCC 中间文件（`*.ihx`、`*.asm`、`*.lst` 等）。仓库中提交的 `firmware.hex` 由本目录源码构建；中间构建产物已被 git 忽略。

## 刷写

如果需要，先安装 `stcgal`：

```bash
python3 -m pip install --user stcgal
```

然后用与 STC15 MCU 对应的正确串口适配器/端口刷写：

```bash
make flash
# 或：
stcgal -p /dev/ttyUSB0 firmware.hex
```

如果 ISP 进入失败，先启动 `stcgal`，再对 MCU/板子上电复位。

建议**飞线**后，使用 STC 官方提供的 STC-ISP 工具刷写该固件。我们尝试了 `stcgal` 等开源工具，但最终得到 Protocol Error，未能成功刷写。

## 注意事项

这个固件刻意做得很精简，并且是板级专用的。在把它刷到不同 ZC-328 版本之前，请先确认实际的 PCB 走线。
