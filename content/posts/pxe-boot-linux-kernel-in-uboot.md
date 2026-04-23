+++
date = '2024-09-06T10:33:32+08:00'
draft = false
title = 'Booting a Linux Kernel via PXE in U-Boot'
tags = ["linux", "riscv"]
summary = 'How to set up TFTP-based PXE boot in U-Boot to load a Linux kernel and device tree over the network, including the fdt_addr_r pitfall on SpacemiT K1.'
+++

## Environment

- Board: BPI-F3 (SpacemiT K1)
- Prerequisite: TFTP server already running on the host machine

## Setting Up IP Addresses

Find the host IP:

```bash
ifconfig
# Look for the interface connected to the board
# e.g. inet 192.168.230.28
```

In U-Boot, set the board and server IPs:

```bash
setenv ipaddr 192.168.230.4
setenv serverip 192.168.230.28
saveenv
```

Make sure both are on the same subnet.

## Configuring PXE Files

On the host, set up the TFTP directory:

```
.
├── Image
├── k1-x_deb1.dtb
└── pxelinux.cfg
    └── 01-fe-fe-fe-81-b4-a8
```

- `Image` — the Linux kernel image
- `k1-x_deb1.dtb` — the device tree blob
- `pxelinux.cfg/` — directory containing PXE config files

### Config File Naming

PXE looks up config files by a priority order based on U-Boot variables. The highest priority is the MAC address. Check it in U-Boot:

```bash
printenv ethaddr
# ethaddr=FE:FE:FE:81:B4:A8
```

The config filename is the MAC address in lowercase, separated by hyphens, prefixed with `01-` (the ARP hardware type for Ethernet):

```
01-fe-fe-fe-81-b4-a8
```

### Config File Content

The format is identical to extlinux.conf:

```
default linux

label linux
	kernel Image
	fdt k1-x_deb1.dtb
	append earlycon=sbi earlyprintk console=ttyS0,115200 loglevel=8 clk_ignore_unused swiotlb=65536 rdinit=/init workqueue.default_affinity_scope=system root=/dev/mmcblk2p6 rootwait rootfstype=ext4
```

## Booting

In U-Boot:

```bash
pxe get
pxe boot
```

## Pitfall: Missing fdt_addr_r

If the kernel hangs during boot, check the log for something like:

```
## Flattened Device Tree blob at 7deb2e10
   Booting using the fdt blob at 0x7deb2e10
```

This address (`0x7deb2e10`) is not where your DTB was loaded — it's `fdtcontroladdr`, U-Boot's own internal device tree. This means PXE didn't know where to load the DTB you specified.

PXE loads the kernel to `kernel_addr_r` and the DTB to `fdt_addr_r`. On this board, `kernel_addr_r` was set but `fdt_addr_r` was not:

```bash
printenv kernel_addr_r
# kernel_addr_r=0x11000000

printenv fdt_addr_r
# Error: "fdt_addr_r" not defined
```

The fix:

```bash
setenv fdt_addr_r 0x31000000
saveenv
```

After this, `pxe get && pxe boot` works correctly — the DTB is loaded to the right address and the kernel boots with the proper device tree.

## Reference

- [CSDN: PXE boot in U-Boot](https://blog.csdn.net/weixin_35808698/article/details/117274748)
