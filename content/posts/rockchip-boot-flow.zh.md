+++
date = '2024-08-23T12:57:43+08:00'
draft = false
title = 'Rockchip 系列启动流程解读'
tags = ["linux"]
summary = '详解 Rockchip SoC 从 BootROM 到 U-Boot 的完整启动流程，涵盖开源 TPL/SPL 路径和闭源 miniloader 路径。'
+++

## SoC 启动流程概述

Rockchip SoC 上电后，第一段执行的代码是 **BootROM** — 在芯片制造时固化在片内 ROM 中的程序，不可修改。

BootROM 的职责：
- 初始化 CPU（模式、缓存、时钟）
- 关闭看门狗
- 初始化存储控制器（MMC、NAND、SPI NOR、USB），以便从外部存储加载下一级引导程序

硬件初始化完成后，BootROM 需要从外部存储加载下一段可执行程序到内存。但此时 DRAM 尚未初始化 — DDR 控制器需要显式的训练和配置。因此 BootROM 将一小段程序加载到芯片内部的 **SRAM**（通常 64–256 KB，取决于具体 SoC）。这段程序的唯一任务就是初始化 DRAM。

DRAM 就绪后，执行流返回 BootROM，BootROM 再加载一段更大的程序到 DRAM。这段程序负责将 U-Boot 和 ARM Trusted Firmware（ATF）加载到 DRAM 并跳转执行。

### ARM Trusted Firmware（ATF）

Rockchip ARM64 SoC（RK3399、RK3568、RK3588 等）需要 ATF（也叫 BL31）。ATF 运行在最高特权级 EL3，负责：
- 在跳转到 U-Boot 之前将 CPU 从安全的 EL3 切换到 EL2
- PSCI（Power State Coordination Interface）— 内核启动后负责唤醒其他 CPU 核心
- 运行时安全服务

---

## 开源路径：TPL + SPL

在完全开源的启动流程中：

- **TPL（Tertiary Program Loader）** — 从 SRAM 运行，初始化 DRAM
- **SPL（Secondary Program Loader）** — 从 DRAM 运行，加载 U-Boot 和 ATF

### 在 U-Boot 中启用 TPL/SPL

在 Rockchip U-Boot defconfig 中，关键选项：

```
CONFIG_SUPPORT_SPL=y
CONFIG_SUPPORT_TPL=y
CONFIG_SPL=y
CONFIG_TPL=y
CONFIG_SPL_ATF=y
CONFIG_TPL_TINY_FRAMEWORK=y
```

完整的 SPL/TPL 配置还包括 MMC 支持、加密、MTD 等众多选项 — 具体参考对应板子的 defconfig。

编译后生成的关键文件：

```
tpl/u-boot-tpl.bin    # DRAM 初始化（从 SRAM 运行）
spl/u-boot-spl.bin    # 加载 U-Boot + ATF（从 DRAM 运行）
```

### 打包为 idbloader.img

BootROM 要求特定的镜像格式，带有 **ID Block header**。使用 `mkimage` 生成：

```bash
# 创建带 ID Block header 的 idbloader（包含 TPL）
tools/mkimage -n rk3399 -T rksd -d tpl/u-boot-tpl.bin idbloader.img

# 追加 SPL
cat spl/u-boot-spl.bin >> idbloader.img
```

将 `rk3399` 替换为你的 SoC 型号（如 `rk3568`、`rk3588`）。

生成的 `idbloader.img` 写入启动介质的 sector 64（0x40）— 这是 BootROM 查找引导程序的位置。

---

## 闭源路径：Rockchip Miniloader

在厂商启动流程中，Rockchip 通过 [rkbin 仓库](https://github.com/rockchip-linux/rkbin) 提供预编译的二进制文件：

### 第一步：打包 idbloader.img

```bash
tools/mkimage -n rk3568 -T rksd -d rkbin/bin/rk35/rk3568_ddr_1560MHz_v1.xx.bin idbloader.img
cat rkbin/bin/rk35/rk356x_spl_v1.xx.bin >> idbloader.img
```

其中：
- `rk3568_ddr_*.bin` — 等价于 TPL，负责初始化 DDR
- `rk356x_spl_*.bin` — Rockchip 的闭源 miniloader，等价于 SPL

### 第二步：打包 u-boot.img

miniloader 要求 U-Boot 使用特定格式。用 `loaderimage` 转换：

```bash
tools/loaderimage --pack --uboot u-boot.bin u-boot.img 0x200000
```

这会将 U-Boot 源码编译出的 `u-boot.bin` 包装成 miniloader 可识别的格式，并指定加载地址。

### 第三步：打包 trust.img

ATF（BL31）同样需要转换为 miniloader 格式：

```bash
tools/trust_merger trust.ini
```

其中 `trust.ini` 引用从 ARM Trusted Firmware 源码编译的 `bl31.elf`（或 rkbin 中预编译的 `bl31.bin`）。

---

## 开源 vs 闭源对比

| 组件 | 开源 | 闭源 |
|------|------|------|
| DRAM 初始化 | TPL (`u-boot-tpl.bin`) | `rk35xx_ddr_*.bin`（二进制 blob） |
| 加载器 | SPL (`u-boot-spl.bin`) | `rk35xx_spl_*.bin`（二进制 blob） |
| U-Boot 格式 | FIT image（标准格式） | `loaderimage` 封装 |
| ATF 格式 | SPL 直接加载 | `trust_merger` 封装 |
| 可调试性 | 完整源码，可加 print | 黑盒 |

开源路径适合开发和上游贡献。闭源路径是 Rockchip 生产 SDK 中使用的方式，可能包含开源 TPL 中没有的额外 DDR 训练优化。

## 参考

- [Rockchip Boot Flow（rkbin 文档）](https://github.com/rockchip-linux/rkbin)
- [U-Boot Rockchip 文档](https://docs.u-boot.org/en/latest/board/rockchip/)
