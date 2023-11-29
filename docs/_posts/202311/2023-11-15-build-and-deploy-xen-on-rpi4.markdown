---
layout: post
title: 编译XEN并部署在Raspberry PI 4上
date: 2023-11-15
tags: [hypervisor, Raspberry PI]
categories: [hypervisor, Raspberry PI]
---

Raspberry PI 4上的中断控制器使用了ARM的标准的GIC-400中断控制器，使得XEN可以在Raspberry PI 4上运行。

## 为Raspberry PI4编译XEN

当前XEN最新的Release版本是``RELEASE-4.17.2``。

### 下载Source Code

从Github上下载最新的Code，并切到最新的Release分支上：

```bash
$ gh repo clone xen-project/xen
...
$ cd xen

$ checkout RELEASE-4.17.2
```

### 编译

现在XEN的编译用了类似Linux Kernel的Makefile架构，使用``make menuconfig``的方式进行编译配置，不再支持在``make``后面加上参数``CONFIG_XXX=xxx``的方式了。

为了以后修改方便，可以创建一个针对Raspberry PI 4的配置文件，放在``xen/arch/arm/configs/``目录下，命名为``rpi4_defconfig``文件，内容如下：

```
CONFIG_DEBUG=y
CONFIG_SCHED_ARINC653=y
CONFIG_EARLY_UART_CHOICE_8250=y
CONFIG_EARLY_UART_BASE_ADDRESS=0xfe215040
CONFIG_EARLY_UART_8250_REG_SHIFT=2
```

套用``rpi4_defconfig``的配置：

```bash
$ make -C xen XEN_TARGET_ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- rpi4_defconfig
```

编译：

```bash
$ make -j8 XEN_TARGET_ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- dist-xen
```

如果没有出错的话，会在``xen``的目录下生成一个名字为``xen``的文件。

### 验证

把生成的``xen``放到Raspberry PI 板子上存储空间的引导分区``boot``里。

在U-Boot启动时中断启动，并把``xen``作为kernel引导：

```
...
Hit any key to stop autoboot:  0
U-Boot> fatload usb 0:1 ${kernel_addr_r} xen
1146888 bytes read in 7 ms (156.3 MiB/s)
U-Boot> booti ${kernel_addr_r} - ${fdt_addr}
Moving Image from 0x80000 to 0x200000, end=35e0c8
## Flattened Device Tree blob at 2df9e600
   Booting using the fdt blob at 0x2df9e600
Working FDT set to 2df9e600
   Using Device Tree in place at 000000002df9e600, end 000000002dfaefec
Working FDT set to 2df9e600

Starting kernel ...

- UART enabled -
- Boot CPU booting -
- Current EL 0000000000000008 -
- Initialize CPU -
- Turning on paging -
- Zero BSS -
- Ready -
(XEN) Checking for initrd in /chosen
(XEN) Initrd 000000002dfac000-000000002efff165
(XEN) RAM: 0000000000000000 - 000000003b2fffff
(XEN) RAM: 0000000040000000 - 00000000fbffffff
(XEN) RAM: 0000000100000000 - 000000017fffffff
(XEN) RAM: 0000000180000000 - 00000001ffffffff
(XEN)
(XEN) MODULE[0]: 0000000000200000 - 000000000035e0c8 Xen
(XEN) MODULE[1]: 000000002df9e600 - 000000002dfad000 Device Tree
(XEN) MODULE[2]: 000000002dfac000 - 000000002efff165 Ramdisk
(XEN)  RESVD[0]: 0000000000000000 - 0000000000001000
(XEN)  RESVD[1]: 000000003ee63d60 - 000000003ee63dac
(XEN)  RESVD[2]: 000000003ee63920 - 000000003ee63d1f
(XEN)
(XEN)
(XEN) Command line: <NULL>
(XEN) Domain heap initialised
(XEN) Booting using Device Tree
(XEN) Platform: Raspberry Pi 4
...
(XEN) *** LOADING DOMAIN 0 ***
(XEN) Missing kernel boot module?
(XEN)
(XEN) ****************************************
(XEN) Panic on CPU 0:
(XEN) Could not set up DOM0 guest OS
(XEN) ****************************************
```

可以看到XEN以及启动。但是现在还没有配置XEN的启动参数，以及DOM0的参数，所以会启动失败。

## 部署XEN

### Dom0 Kernel

下载代码：

```bash
$ git clone --depth=1 --branch rpi-6.1.y https://github.com/raspberrypi/linux
```

配置：


添加XEN相关配置：

```bash
$ KERNEL=kernel8 make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- bcm2711_defconfig xen.config
```

编译：

```bash
$ KERNEL=kernel8 make -j8 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- Image modules dtbs
```

部署：

```bash
$ make -j8 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- INSTALL_MOD_PATH=../rootfs modules_install
```

Kernel位置：``arch/arm64/boot/Image``，Modules位置：``../rootfs/lib/modules/``。

把Dom0的Kernel复制到启动分区``boot``下面，把Modules复制到根分区``root``下的``/lib/modules/``下。

### 启动文件

创建启动文件``boot_xen.source``，内容如下:

```
setenv xen_addr 0x00E00000
setenv dom0_kernel_addr 0x01000000
setenv dom0_kernel_size 0x01780000
fatload usb 0:1 ${xen_addr} xen
fatload usb 0:1 ${dom0_kernel_addr} kernel8-dom0.img
fatload usb 0:1 ${fdt_addr} bcm2711-rpi-4-b.dtb

fdt addr ${fdt_addr}
fdt resize 1024
fdt set /chosen \#address-cells <0x2>
fdt set /chosen \#size-cells <0x2>
fdt set /chosen xen,xen-bootargs "console=dtuart dtuart=serial0 sync_console dom0_mem=4G dom0_max_vcpus=2 bootscrub=0 vwfi=native sched=null"
fdt mknod /chosen dom0
fdt set /chosen/dom0 compatible "xen,linux-zimage" "xen,multiboot-module"
fdt set /chosen/dom0 reg <0x0 ${dom0_kernel_addr} 0x0 ${dom0_kernel_size}>
fdt set /chosen xen,dom0-bootargs "coherent_pool=1M 8250.nr_uarts=1 snd_bcm2835.enable_compat_alsa=0 snd_bcm2835.enable_hdmi=1 snd_bcm2835.enable_headphones=1 video=HDMI-A-1:1920x1080M@60 smsc95xx.macaddr=DC:A6:32:D1:9D:BF vc_mem.mem_base=0x3eb00000 vc_mem.mem_size=0x3ff00000 dwc_otg.lpm_enable=0 console=ttyAMA0 console=hvc0 earlycon=xen earlyprintk=xen root=/dev/sda2 rootfstype=ext4 elevator=deadline rootwait fixrtc splash"
setenv fdt_high 0xffffffffffffffff
booti ${xen_addr} - ${fdt_addr}
```

然后用以下的命令编译生成``boot_xen.scr``文件，并复制到启动分区``boot``下：

```bash
$ mkimage -A arm64 -T script -C none -a 0xC00000 -e 0xC00000 -d boot_xen.source boot_xen.scr
$ sudo cp boot_xen.scr /boot/firmware
```

### 启动XEN + Dom0

在U-Boot启动过程中，中断自动启动，然后手动加载``boot_xen.scr``文件，并启动：

```
U-Boot> fatload usb 0:1 0xC00000 boot_xen.scr
1334 bytes read in 2 ms (651.4 KiB/s)
U-Boot> source 0xC00000
...
```

查看CPU信息（Dom0分配了2个vCPU）：

```
$ cat /proc/cpuinfo
processor	: 0
BogoMIPS	: 108.00
Features	: fp asimd evtstrm crc32 cpuid
CPU implementer	: 0x41
CPU architecture: 8
CPU variant	: 0x0
CPU part	: 0xd08
CPU revision	: 3

processor	: 1
BogoMIPS	: 108.00
Features	: fp asimd evtstrm crc32 cpuid
CPU implementer	: 0x41
CPU architecture: 8
CPU variant	: 0x0
CPU part	: 0xd08
CPU revision	: 3

Serial		: XXXXXXXXX
Model		: Raspberry Pi 4 Model B
```

内存信息（Dom0分配了4GB的内存）：

```
$ free -h
               total        used        free      shared  buff/cache   available
Mem:           3.8Gi       207Mi       3.4Gi       1.3Mi       344Mi       3.6Gi
Swap:           99Mi          0B        99Mi
```

### 添加XEN相关Modules

创建新文件``/usr/lib/modules-load.d/xen.conf``：

```
xen-evtchn
xen-gntdev
xen-gntalloc
xen-blkback
xen-netback
```

## 问题

- BT/WiFi：蓝牙不能使用，WiFi暂时没有发现问题
- Graphics：图形界面不能使用
- ``reboot``：命令不能重启

## 参考

- [1][XEN Project: XEN ON RASPBERRY PI 4 ADVENTURES](https://xenproject.org/2020/09/29/xen-on-raspberry-pi-4-adventures/)
- [2][XEN Wiki: ImageBuilder](https://wiki.xenproject.org/wiki/ImageBuilder)
- [3][Raspberry Pi Forum: Installing Xen on Raspberry Pi](https://forums.raspberrypi.com/viewtopic.php?t=232323)
