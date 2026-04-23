+++
date = '2024-08-23T12:57:43+08:00'
draft = false
title = 'Rockchip Boot Flow Explained'
tags = ["linux"]
summary = 'A walkthrough of the Rockchip SoC boot sequence from BootROM to U-Boot, covering both the open-source TPL/SPL path and the closed-source miniloader path.'
+++

## SoC Boot Flow Overview

When a Rockchip SoC powers on, the first code to execute is the **BootROM** — a small piece of firmware baked into the chip at manufacturing time. It is immutable and lives in on-chip ROM.

BootROM responsibilities:
- Initialize the CPU (mode, caches, clocks)
- Disable the watchdog
- Initialize storage controllers (MMC, NAND, SPI NOR, USB) to locate the next-stage bootloader

After hardware init, BootROM needs to load the next executable from external storage into memory. But DRAM hasn't been initialized yet — the DDR controller requires explicit training and configuration. So BootROM loads a small program into the chip's internal **SRAM** (typically 64–256 KB, depending on the SoC). This program's sole job is to bring up DRAM.

Once DRAM is ready, execution returns to BootROM, which then loads a larger program into DRAM. This program is responsible for loading U-Boot proper and the ARM Trusted Firmware (ATF) into DRAM and transferring control to them.

### ARM Trusted Firmware (ATF)

Rockchip ARM64 SoCs (RK3399, RK3568, RK3588, etc.) require ATF (also called BL31). ATF runs at the highest privilege level (EL3) and handles:
- Transitioning the CPU from secure EL3 to EL2 before jumping to U-Boot
- PSCI (Power State Coordination Interface) — bringing secondary CPU cores online after the kernel boots
- Runtime secure services

---

## Open-Source Path: TPL + SPL

In the fully open-source boot flow:

- **TPL (Tertiary Program Loader)** — runs from SRAM, initializes DRAM
- **SPL (Secondary Program Loader)** — runs from DRAM, loads U-Boot proper + ATF

### Enabling TPL/SPL in U-Boot

In the Rockchip U-Boot defconfig, the key options are:

```
CONFIG_SUPPORT_SPL=y
CONFIG_SUPPORT_TPL=y
CONFIG_SPL=y
CONFIG_TPL=y
CONFIG_SPL_ATF=y
CONFIG_TPL_TINY_FRAMEWORK=y
```

A full SPL/TPL config includes many more options (MMC support, crypto, MTD, etc.) — refer to the board's defconfig for the complete list.

After building, the relevant binaries are:

```
tpl/u-boot-tpl.bin    # DRAM init (runs from SRAM)
spl/u-boot-spl.bin    # Loads U-Boot + ATF (runs from DRAM)
```

### Packaging into idbloader.img

BootROM expects a specific image format with an **ID Block header**. Use `mkimage` to create it:

```bash
# Create idbloader with TPL (adds the ID Block header)
tools/mkimage -n rk3399 -T rksd -d tpl/u-boot-tpl.bin idbloader.img

# Append SPL
cat spl/u-boot-spl.bin >> idbloader.img
```

Replace `rk3399` with your SoC model (e.g., `rk3568`, `rk3588`).

The resulting `idbloader.img` is written to sector 64 (0x40) of the boot media — this is where BootROM looks for it.

---

## Closed-Source Path: Rockchip Miniloader

In the vendor boot flow, Rockchip provides prebuilt binaries via the [rkbin repository](https://github.com/rockchip-linux/rkbin):

### Step 1: Package idbloader.img

```bash
tools/mkimage -n rk3568 -T rksd -d rkbin/bin/rk35/rk3568_ddr_1560MHz_v1.xx.bin idbloader.img
cat rkbin/bin/rk35/rk356x_spl_v1.xx.bin >> idbloader.img
```

Where:
- `rk3568_ddr_*.bin` — equivalent to TPL, initializes DDR
- `rk356x_spl_*.bin` — Rockchip's proprietary miniloader, equivalent to SPL

### Step 2: Package u-boot.img

The miniloader expects U-Boot in a specific format. Use `loaderimage` to convert:

```bash
tools/loaderimage --pack --uboot u-boot.bin u-boot.img 0x200000
```

This wraps `u-boot.bin` (compiled from U-Boot source) in the miniloader's expected format with a load address.

### Step 3: Package trust.img

ATF (BL31) also needs to be in miniloader format:

```bash
tools/trust_merger trust.ini
```

Where `trust.ini` references the `bl31.elf` compiled from ARM Trusted Firmware source (or the prebuilt `bl31.bin` from rkbin).

---

## Open-Source vs Closed-Source Comparison

| Component | Open-Source | Closed-Source |
|-----------|------------|---------------|
| DRAM init | TPL (`u-boot-tpl.bin`) | `rk35xx_ddr_*.bin` (blob) |
| Loader | SPL (`u-boot-spl.bin`) | `rk35xx_spl_*.bin` (blob) |
| U-Boot format | FIT image (standard) | `loaderimage` wrapped |
| ATF format | Loaded by SPL directly | `trust_merger` wrapped |
| Debuggability | Full source, can add prints | Black box |

The open-source path is preferred for development and upstream work. The closed-source path is what Rockchip uses in production SDKs and may include additional DDR training optimizations not available in the open-source TPL.

## References

- [Rockchip Boot Flow (rkbin docs)](https://github.com/rockchip-linux/rkbin)
- [U-Boot Rockchip documentation](https://docs.u-boot.org/en/latest/board/rockchip/)
