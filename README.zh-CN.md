# Armbian - ZC-328

[English](README.md) | **简体中文**

ZC-328 主板的 Armbian 补丁（这是一个第三方爱好者项目，非官方作品）。

本仓库包含 board config、U-Boot 补丁、Linux DTS 补丁、kernel config、该板子上使用的 STC15 辅助固件，以及一个可以自动编译并发布开箱即用镜像的 CI 工作流。

## 目录

- [已验证基线](#已验证基线)
- [CI 预编译镜像](#ci-预编译镜像)
- [仓库结构](#仓库结构)
- [使用说明](#使用说明)
- [烧录到 eMMC](#烧录到-emmc)
- [关于 STC15 的固件](#关于-stc15-的固件)
- [软件许可](#软件许可)

## 已验证基线

本次移植与实机验证基于：

- Armbian build tag：`v26.8.0-trunk.53`（本地实测）
- Armbian build commit：`9bd9bcd090a2e1ff385ec161ea4c4633273e2d19`
- 本地构建分支：`zc328-v26.8`
- U-Boot：`v2026.04` 补丁
- Linux kernel family：Rockchip current / Linux `6.18`
- 实测 kernel：`6.18.34-current-rockchip`
- 实测镜像：`Armbian-unofficial_26.05.0-trunk_Zc-328_noble_current_6.18.34.img`
- 发行版：Ubuntu `noble`

> CI 工作流当前基于 `v26.11.0-trunk.30` 构建。已对照该 tag 重新评估，移植可近乎原样套用（与已验证基线相同的 Linux 6.18 kernel family 和 U-Boot v2026.04）。

## CI 预编译镜像

[CI 工作流](.github/workflows/build.yml) 在 `v26.11.0-trunk.30` 上用与 README 基线一致的参数编译 ZC-328 镜像，并把产物发布为 GitHub release：

- Release 名称：`CI <armbian tag> <编译日期 (UTC+0)>`
- 产物：xz 压缩镜像（`.img.xz`）+ SHA256 校验文件 + deb 软件包
- 在 **Actions** 页手动触发（可选指定 Armbian tag）
- Release 描述以表格列出全部产物，并写明默认登录账号

默认登录账号（Armbian 标准默认值）：

```text
用户名：root
密码：1234   （首次登录会强制要求修改）
```

## 仓库结构

```text
config/boards/zc-328.conf                       板级定义
config/kernel/linux-rockchip-current.config     内核配置（current 分支）
config/kernel/linux-rockchip-current.config.defconfig
config/sources/families/rockchip.conf           rockchip family 配置（含 zc-328 块）
userpatches/kernel/archive/rockchip-6.18/       Linux 6.18 补丁
  - 0001-zc328-rk3288-evb-rk808-dts.patch       ZC-328 走线的 DTS 修正
  - 0002-zc328-usb-otg-host.patch               强制 USB OTG 为 host 模式
userpatches/u-boot/v2026.04/                    U-Boot v2026.04 补丁
  - 00-zc328-ddr-timing-v2026.04.patch          DDR 初始化时序
  - 02-zc328-rk8xx-defconfig-v2026.04.patch     RK8xx defconfig 调整
  （v2022.04/ 是旧的 U-Boot 补丁，已不再使用）
mcu/stc15-rk808-reset-release/                  STC15 辅助固件（见下文）
.github/workflows/build.yml                     CI 编译与发布工作流
```

## 使用说明

1. 将本项目克隆到本地（或者下载 zip 文件）。
2. 将 [armbian 项目](https://github.com/armbian/build) 克隆到本地，切换到 `v26.8.0-trunk.53` tag（对应 commit：`9bd9bcd090a2e1ff385ec161ea4c4633273e2d19`），或切换到上面提到的任何你想编译的 tag。
3. 将本项目中的所有文件复制粘贴到 armbian 目录下（例如 `/home/username/armbian/build`）。
4. 运行 armbian 的编译脚本来编译你自己的镜像：

```bash
   ./compile.sh build \
     BOARD=zc-328 \
     BRANCH=current \
     BUILD_DESKTOP=no \
     BUILD_MINIMAL=no \
     KERNEL_CONFIGURE=no \
     KERNEL_GIT=full \
     NO_HOST_RELEASE_CHECK=yes \
     RELEASE=noble
```

5. 编译完成后，你可以在 `output/images` 目录下找到你的镜像（例如 `/home/username/armbian/build/output/images`）。
6. 将镜像烧录到 SD 卡，大功告成！

## 烧录到 eMMC

你可以使用 `rkdeveloptool` 来烧录镜像到 eMMC。`rkdeveloptool` 已经包含在 armbian 的编译脚本中，所以你可以直接使用：

```
# 0. 将主板进入 maskrom 模式
#    - 通过 USB OTG 口将主板连接到电脑
#    - 然后按住 EMMC 芯片旁边的 "uboot" 按钮（这一步非常重要！）
#    - 最后连接 12V 电源

# 1. 下载 uboot
/home/username/armbian/build/cache/sources/rkbin-tools/tools/rkdeveloptool db /home/username/armbian/build/cache/sources/rkbin-tools/rk32/rk3288_ubootloader_v1.01.06.bin

# 2. 烧录镜像到 EMMC
/home/username/armbian/build/cache/sources/rkbin-tools/tools/rkdeveloptool wl 0 /home/username/armbian/build/output/images/你的镜像文件.img

# 3. 重启主板
/home/username/armbian/build/cache/sources/rkbin-tools/tools/rkdeveloptool rd

# 4. 搞定，大功告成！
```

写入 eMMC 后，**强烈建议** hold 住板级 kernel/DTB/U-Boot 包，避免普通升级覆盖自定义移植而导致无法启动：

```bash
apt-mark hold \
  linux-image-current-rockchip \
  linux-dtb-current-rockchip \
  linux-u-boot-zc-328-current \
  linux-headers-current-rockchip \
  linux-libc-dev-current-rockchip \
  armbian-bsp-cli-zc-328-current
```

## 关于 STC15 的固件

ZC-328 广告屏板子上有一个 STC15/8051 小 MCU，它的原设计作用是控制 RK808 PMIC 的 reset 相关网络，实现了看门狗和红外、定时开关机等功能，并通过 RK3288 UART 3 与 RK3288 通讯。

该项目暂时没有办法读出该 MCU 的原始固件，因此无法得知如何进行看门狗喂狗。如果没有正确的喂狗，该 STC15 MCU 将会大约 40s 重启一次 RK808 PMIC，严重干扰正常使用。

因此 **重新刷写 STC15 的固件是十分必要的**。

通过测量板子上的连接，我们发现 STC15 MCU 的 `P1.3` 引脚连接到了 RK808 PMIC 的 `VPPOTP` 引脚。而查阅 RK808 PMIC 的 datasheet，我们发现 `VPPOTP` 在官方参考设计中被用作高电平复位，大致连接如下：

```text
VCCRTC -- 10k -- RESET button -- VPPOTP
```

所以本仓库实现了一个最简化的 STC15 固件，该固件会把 `P1.3` 配置成高阻输入，然后让 STC15 进入 power-down 模式，借此尽量模拟 reset 按钮未按下的状态。代码位于 `mcu/stc15-rk808-reset-release/` 目录（详见[该目录的 README](mcu/stc15-rk808-reset-release/README.md)）：

- `main.c` - 最小 reset-release 固件源码
- `stc15.h` - SDCC 所需 STC15 SFR 定义
- `Makefile` - SDCC/packihx 构建规则
- `firmware.hex` - 预构建 Intel HEX

预构建 `firmware.hex` 的 sha256：

```text
5f8dc69edb56a1263b67e879ccf881bb6596b2502c85cd48847909888f07ae5b
```

构建该固件：

```bash
cd mcu/stc15-rk808-reset-release
sudo apt install sdcc
make
```

建议**飞线**后，使用 STC 官方提供的 STC-ISP 工具刷写该固件。我们尝试了 `stcgal` 等开源工具，但最终得到 Protocol Error，未能成功刷写。

## 软件许可

本项目基于 GPLv2 协议发布，没有任何担保。更多详情请参阅 [LICENSE](LICENSE) 文件。

本项目是一个爱好者项目，与 zc-328 主板的原始制造商无关。请不要联系原始制造商寻求技术支持。
