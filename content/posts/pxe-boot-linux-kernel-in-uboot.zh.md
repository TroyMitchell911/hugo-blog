+++
date = '2024-09-06T10:33:32+08:00'
draft = false
title = '使用 PXE 在 U-Boot 中启动 Linux 内核'
tags = ["linux", "riscv"]
summary = '通过 TFTP 配置 PXE 启动，在 U-Boot 中加载 Linux 内核和设备树，包括 SpacemiT K1 上 fdt_addr_r 缺失导致启动卡死的排查过程。'
+++

## 环境

- 开发板：BPI-F3（SpacemiT K1）
- 前提：主机已搭建好 TFTP 服务

## 设置 IP 地址

在主机上查询 IP：

```bash
ifconfig
# 找到连接开发板的网口
# 例如 inet 192.168.230.28
```

在 U-Boot 中设置开发板和服务器 IP：

```bash
setenv ipaddr 192.168.230.4
setenv serverip 192.168.230.28
saveenv
```

确保两者在同一子网内。

## 配置 PXE 文件

在主机的 TFTP 目录下准备以下文件：

```
.
├── Image
├── k1-x_deb1.dtb
└── pxelinux.cfg
    └── 01-fe-fe-fe-81-b4-a8
```

- `Image` — Linux 内核镜像
- `k1-x_deb1.dtb` — 设备树文件
- `pxelinux.cfg/` — PXE 配置文件目录

### 配置文件命名

PXE 按优先级查找配置文件，最高优先级是 MAC 地址。在 U-Boot 中查看：

```bash
printenv ethaddr
# ethaddr=FE:FE:FE:81:B4:A8
```

配置文件名为 MAC 地址小写、用连字符分隔，前缀 `01-`（Ethernet 的 ARP 硬件类型）：

```
01-fe-fe-fe-81-b4-a8
```

### 配置文件内容

格式与 extlinux.conf 完全一致：

```
default linux

label linux
	kernel Image
	fdt k1-x_deb1.dtb
	append earlycon=sbi earlyprintk console=ttyS0,115200 loglevel=8 clk_ignore_unused swiotlb=65536 rdinit=/init workqueue.default_affinity_scope=system root=/dev/mmcblk2p6 rootwait rootfstype=ext4
```

## 启动

在 U-Boot 中执行：

```bash
pxe get
pxe boot
```

## 踩坑：fdt_addr_r 缺失

如果内核启动过程中卡死，检查日志中是否出现类似：

```
## Flattened Device Tree blob at 7deb2e10
   Booting using the fdt blob at 0x7deb2e10
```

这个地址（`0x7deb2e10`）并不是你指定的 DTB 加载地址，而是 `fdtcontroladdr` — U-Boot 自身的内部设备树。这说明 PXE 不知道该把 DTB 加载到哪里。

PXE 将内核加载到 `kernel_addr_r`，将 DTB 加载到 `fdt_addr_r`。在这块板子上，`kernel_addr_r` 已设置，但 `fdt_addr_r` 没有：

```bash
printenv kernel_addr_r
# kernel_addr_r=0x11000000

printenv fdt_addr_r
# Error: "fdt_addr_r" not defined
```

修复：

```bash
setenv fdt_addr_r 0x31000000
saveenv
```

之后 `pxe get && pxe boot` 就能正常工作了 — DTB 被加载到正确的地址，内核使用正确的设备树启动。

## 参考

- [CSDN: U-Boot PXE 启动](https://blog.csdn.net/weixin_35808698/article/details/117274748)
