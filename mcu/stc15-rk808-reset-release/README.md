# STC15 RK808 reset-release helper firmware

**English** | [简体中文](README.zh-CN.md)

This directory contains a small SDCC-buildable STC15/8051 firmware used on the
ZC-328 advertising-board variant that has an STC15 MCU connected to the RK808
PMIC `VPPOTP` / reset-related net.

## Table of contents

- [Why this exists](#why-this-exists)
- [How it works](#how-it-works)
- [Repository layout](#repository-layout)
- [Build](#build)
- [Flashing](#flashing)
- [Notes](#notes)

## Why this exists

The original STC firmware appeared to drive `P1.3` / `RK808_VPP_OTP` high and
then enter power-down. On the RK808 reference reset circuit, `VPPOTP` is also
used as a high-active reset input:

```text
VCCRTC -- 10k -- RESET button -- VPPOTP
```

So holding this signal high is effectively like holding the RK808 reset button
pressed, which can prevent the RK3288 board from running normally.

There is currently no way to read out the original MCU firmware, and the
watchdog it implements would otherwise reboot the RK808 PMIC roughly every 40
seconds. Re-flashing this helper firmware is therefore **required** for normal
use of the board.

## How it works

The replacement firmware releases that signal instead of asserting it:

- `P1.3` port latch is released (`P1_3 = 1`) before switching the pin mode.
- `P1.3` is configured as a high-impedance input via `P1M1`/`P1M0`
  (`10` = high-impedance input on STC15).
- The MCU then enters STC power-down.

This tries to emulate the reset button being unpressed, and avoids both holding
`VPPOTP` high and fighting the physical reset-button path through the 10k
resistor. See `main.c` for the exact bit operations.

## Repository layout

- `main.c` - the minimal reset-release firmware source code
- `stc15.h` - the STC15 SFR definitions required by SDCC (`P1M1`/`P1M0` at
  `0x91`/`0x92`)
- `Makefile` - the build rules for SDCC/packihx, plus a `flash` target
- `firmware.hex` - the pre-built Intel HEX (built from the sources here)

The sha256 of the pre-built `firmware.hex` is:

```text
5f8dc69edb56a1263b67e879ccf881bb6596b2502c85cd48847909888f07ae5b
```

## Build

On Debian/Ubuntu:

```bash
sudo apt install sdcc
make
```

`make clean` removes all SDCC intermediate files (`*.ihx`, `*.asm`, `*.lst`,
etc.). The checked-in `firmware.hex` was built from the sources in this
directory; intermediate build artifacts are git-ignored.

## Flashing

Install `stcgal` if desired:

```bash
python3 -m pip install --user stcgal
```

Then flash with the correct serial adapter/port for the STC15 MCU:

```bash
make flash
# or:
stcgal -p /dev/ttyUSB0 firmware.hex
```

If ISP entry fails, start `stcgal` first and then power-cycle the MCU/board.

It is recommended to use the official STC-ISP tool provided by STC to flash
this firmware after flying wires. We tried open-source tools such as `stcgal`,
but unfortunately got a Protocol Error and flashing was unsuccessful.

## Notes

This firmware is intentionally tiny and board-specific. Verify the actual PCB
net before flashing it to a different ZC-328 variant.
