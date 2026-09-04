# Armbian - ZC-328

**English** | [简体中文](README.zh-CN.md)

Armbian patches for the zc-328 board (third party &amp; hobby project, not official).

This repository contains the board configuration, U-Boot patches, Linux DTS
patch, kernel configuration, a small STC15 helper firmware, and a CI workflow
that builds and publishes ready-to-flash images.

## Table of contents

- [Verified baseline](#verified-baseline)
- [Pre-built images (CI)](#pre-built-images-ci)
- [Repository layout](#repository-layout)
- [How to use](#how-to-use)
- [Flashing to eMMC](#flashing-to-emmc)
- [About the STC15 firmware](#about-the-stc15-firmware)
- [License](#license)

## Verified baseline

This port was brought up and tested with:

- Armbian build tag: `v26.8.0-trunk.53` (locally verified)
- Armbian build commit: `9bd9bcd090a2e1ff385ec161ea4c4633273e2d19`
- Build branch used locally: `zc328-v26.8`
- U-Boot: `v2026.04` patches
- Linux kernel family: Rockchip current / Linux `6.18`
- Tested output kernel: `6.18.34-current-rockchip`
- Tested image name: `Armbian-unofficial_26.05.0-trunk_Zc-328_noble_current_6.18.34.img`
- Tested release: Ubuntu `noble`

> The CI workflow currently builds against `v26.11.0-trunk.30`. The port was
> re-evaluated against that tag and applies nearly unchanged (same Linux 6.18
> kernel family and U-Boot v2026.04 as the verified baseline).

## Pre-built images (CI)

The [CI workflow](.github/workflows/build.yml) builds the ZC-328 image on
`v26.11.0-trunk.30` with the same settings as the README baseline and publishes
the result as a GitHub release:

- Release name: `CI <armbian tag> <build date (UTC+0)>`
- Artifacts: xz-compressed image (`.img.xz`) + SHA256 checksum + deb packages
- Triggered manually from the **Actions** tab (optionally pin an Armbian tag)
- The release body lists all files in a table and documents the default login

Default login (standard Armbian):

```text
username: root
password: 1234   (you will be forced to change it on first login)
```

## Repository layout

```text
config/boards/zc-328.conf                       board definition
config/kernel/linux-rockchip-current.config     kernel config (current branch)
config/kernel/linux-rockchip-current.config.defconfig
config/sources/families/rockchip.conf           rockchip family config (+ zc-328 block)
userpatches/kernel/archive/rockchip-6.18/       Linux 6.18 patches
  - 0001-zc328-rk3288-evb-rk808-dts.patch       DTS fix for the ZC-328 wiring
  - 0002-zc328-usb-otg-host.patch               force USB OTG to host mode
userpatches/u-boot/v2026.04/                    U-Boot v2026.04 patches
  - 00-zc328-ddr-timing-v2026.04.patch          DDR init timing
  - 02-zc328-rk8xx-defconfig-v2026.04.patch     RK8xx defconfig adjustments
  (v2022.04/ holds the older U-Boot patches, no longer used)
mcu/stc15-rk808-reset-release/                  STC15 helper firmware (see below)
.github/workflows/build.yml                     CI build & release workflow
```

## How to use

1. Clone this repo to your local machine (or download the zip file).
2. Clone the [armbian repo](https://github.com/armbian/build) and check out the
   `v26.8.0-trunk.53` tag (`9bd9bcd090a2e1ff385ec161ea4c4633273e2d19` commit),
   or any tag listed above that you want to build.
3. Copy all files in this repo into the armbian directory
   (e.g. `/home/username/armbian/build`).
4. Run the armbian build script to build your own image:

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

5. After the build finishes, find your image in the `output/images` directory
   (e.g. `/home/username/armbian/build/output/images`).
6. Flash the image to an SD card and enjoy it!

## Flashing to eMMC

For flashing to eMMC, you can use `rkdeveloptool` — it is already included in
the armbian build script, so you can use it directly:

```
# 0. Put your board into maskrom mode
#    - Connect your board to PC via the USB OTG port
#    - Press and hold the "uboot" button near the EMMC chip (very important!)
#    - Then connect the 12V power supply

# 1. Download bootloader
/home/username/armbian/build/cache/sources/rkbin-tools/tools/rkdeveloptool db /home/username/armbian/build/cache/sources/rkbin-tools/rk32/rk3288_ubootloader_v1.01.06.bin

# 2. Flash image to EMMC
/home/username/armbian/build/cache/sources/rkbin-tools/tools/rkdeveloptool wl 0 /home/username/armbian/build/output/images/YOUR_IMAGE_FILE.img

# 3. Reboot your board.
/home/username/armbian/build/cache/sources/rkbin-tools/tools/rkdeveloptool rd

# 4. That's all. Enjoy it!
```

After installing to eMMC, it is **strongly recommended** to hold the
board-specific kernel/DTB/U-Boot packages, to avoid accidentally replacing this
custom port with an incompatible generic upgrade:

```bash
apt-mark hold \
  linux-image-current-rockchip \
  linux-dtb-current-rockchip \
  linux-u-boot-zc-328-current \
  linux-headers-current-rockchip \
  linux-libc-dev-current-rockchip \
  armbian-bsp-cli-zc-328-current
```

## About the STC15 firmware

A small STC15/8051 MCU is present on the ZC-328 advertising screen board, which
is originally designed to control the reset-related network of the RK808 PMIC,
implementing watchdog, infrared, timed power on/off, etc., and communicating
with the RK3288 via RK3288 UART 3.

There is currently no way to read out the original firmware of this MCU, so it
is unknown how to feed the watchdog. If the watchdog is not fed properly, the
STC15 MCU will reboot the RK808 PMIC approximately every 40 seconds, which
severely interferes with normal use.

Therefore, **it is necessary to re-flash the firmware of the STC15**.

By measuring the connections on the board, we found that the `P1.3` pin of the
STC15 MCU is connected to the `VPPOTP` pin of the RK808 PMIC. And by referring
to the RK808 PMIC datasheet, we found that `VPPOTP` is used as a high-level
reset in the official reference design, roughly connected as follows:

```text
VCCRTC -- 10k -- RESET button -- VPPOTP
```

This repository implements a minimal STC15 firmware that configures `P1.3` as a
high-impedance input and then puts the STC15 into power-down mode, simulating
the state of the reset button not being pressed as much as possible. The code
is located in the `mcu/stc15-rk808-reset-release/` directory (see
[its README](mcu/stc15-rk808-reset-release/README.md) for details):

- `main.c` - the minimal reset-release firmware source code
- `stc15.h` - the STC15 SFR definitions required by SDCC
- `Makefile` - the build rules for SDCC/packihx
- `firmware.hex` - the pre-built Intel HEX

The sha256 of the pre-built `firmware.hex` is:

```text
5f8dc69edb56a1263b67e879ccf881bb6596b2502c85cd48847909888f07ae5b
```

To build this firmware:

```bash
cd mcu/stc15-rk808-reset-release
sudo apt install sdcc
make
```

It is recommended to use the official STC-ISP tool provided by STC to flash
this firmware after flying wires. We tried open-source tools such as `stcgal`,
but unfortunately we got a Protocol Error and flashing was unsuccessful.

## License

This project is licensed under the GPLv2 License - with ABSOLUTELY NO WARRANTY.
See the [LICENSE](LICENSE) file for details.

This project is a hobby project and has nothing to do with the original
manufacturer of the zc-328 board. Please do not contact the original
manufacturer for technical support.
