---
layout: post
title: Enabling Xen on Yocto for Raspberry Pi 4
date: 2026-03-19
tag: [yocto, raspberrypi, xen]
categories: [Yocto]
---

## Introduction

In this experiment, I explored running the Xen hypervisor on Raspberry Pi 4 using Yocto.

The goal was to build a Yocto image with Xen support, boot it on Raspberry Pi 4, and verify that the system can run with Xen as the hypervisor and Linux as Dom0.

## Environment

- Host: macOS + OrbStack (or Linux)
- Build system: Yocto (scarthgap)
- Target: Raspberry Pi 4 (ARM64)

## Add meta-virtualization Layer

Clone the virtualization layer:

```bash
git clone -b scarthgap https://git.yoctoproject.org/meta-virtualization
```

Add it to the build:

```bash
bitbake-layers add-layer ../meta-openembedded/meta-filesystems
bitbake-layers add-layer ../meta-virtualization
```

## Enable Xen in Yocto

Add or modify file `conf/local.conf`:

```
DISTRO_FEATURES:append = " virtualization xen"

IMAGE_INSTALL:append = " xen xen-tools-xl xen-tools-dev"

RPI_EXTRA_CONFIG = ' \n \
    arm_64bit=1 \n \
    enable_uart=1 \n \
    '

RPI_EXTRA_CONFIG = "force_turbo=1"

IMAGE_BOOT_FILES:append = " xen-raspberrypi4-64;xen"
```

## Build Xen Image

Build a Xen-enabled image:

```bash
bitbake xen-image-minimal
```

Using `bitbake core-image-minimal` resulted in missing Xen-related components such as `gnttab`.

In my setup, `bitbake xen-image-minimal` worked correctly, which suggests that `core-image-minimal` does not automatically include the packages and modules required for Xen.

## Boot Flow on Raspberry Pi 4

The expected boot sequence:

```
Raspberry Pi firmware -> U-Boot -> Xen Hypervisor -> Dom0 Linux
```

Connect UART and observe logs.

Successful boot should show:

```
(XEN) Xen version 4.18.1-pre (bh4bxl@) (aarch64-poky-linux-gcc (GCC) 13.4.0) debug=n 2024-03-12
(XEN) Latest ChangeSet: Mon Mar 4 16:24:21 2024 +0100 git:4da8ca9cb9-dirty
...
(XEN) *** LOADING DOMAIN 0 ***
(XEN) Loading d0 kernel from boot module @ 0000000000400000
(XEN) Allocating 1:1 mappings totalling 256MB for dom0:
(XEN) BANK[0] 0x00000010000000-0x00000020000000 (256MB)
(XEN) Grant table range: 0x00000000200000-0x00000000240000
...
(XEN) *** Serial input to DOM0 (type 'CTRL-a' three times to switch input)
(XEN) Freed 360kB init memory.
Booting Linux on physical CPU 0x0000000000 [0x410fd083]
Linux version 6.6.123-yocto-standard (oe-user@oe-host) (aarch64-poky-linux-gcc (GCC) 13.4.0, GNU ld (GNU Binutils) 2.42.0.20240723) #1 SMP PREEMPT Mon Feb  9 22:44:34 UTC 2026
...
[done]

Poky (Yocto Project Reference Distro) 5.0.16 raspberrypi4-64 hvc0

raspberrypi4-64 login:
```

## Add exFAT Support for Dom0

USB storage is a convenient way to copy the DomU image, kernel, and configuration files. However, the default build does not support exFAT, so I needed to enable exFAT support in the Linux kernel.

### Identify the Active Kernel Recipe

First, I checked which kernel recipe was being used:

```bash
bitbake -e virtual/kernel | grep "^FILE="
```

The result was:

```
FILE="/mnt/yocto/rpi-build/poky/meta/recipes-kernel/linux/linux-yocto_6.6.bb"
```

This shows that the active kernel recipe is based on `linux-yocto`, and the recipe directory is:

```
/mnt/yocto/rpi-build/poky/meta/recipes-kernel/linux/
```

### Initial Attempt: Add a Custom Config Fragment

My first attempt was to add a new config fragment (`exfat.cfg`) through a `.bbappend` file:

```
CONFIG_EXFAT_FS=m
```

And append it via:

```bitbake
SRC_URI:append = " file://exfat.cfg
```

However, after rebuilding, the exFAT module was still missing.

### Debugging the Issue

To understand what was happening, I inspected the effective `SRC_URI`:

```bash
bitbake -e linux-yocto | grep -n 'exfat.cfg\|^SRC_URI='
```

The output showed:

```
SRC_URI="... file://extra-configs.cfg"
```

There was no reference to `exfat.cfg`, which means my fragment was not actually included in the build.

### Solution: Reuse Existing `extra-configs.cfg`

Instead of adding a new fragment, I modified the existing `extra-configs.cfg`, which is already included in the kernel recipe.

I added the following lines:

```
CONFIG_EXFAT_FS=m
CONFIG_NLS=y
CONFIG_NLS_UTF8=y
```

### Rebuild the Kernel

After these changes, the kernel needs to be rebuilt:

```bash
bitbake -c cleansstate virtual/kernel
bitbake -c configure virtual/kernel
bitbake xen-image-minimal
```
After flashing the new image to the SD card and booting the Raspberry Pi 4, I verified that the module was available:

```bash
lsmod
```

Output:

```
Module                  Size  Used by
xen_netback            57344  0
xen_blkback            40960  0
xen_gntalloc           12288  0
xen_gntdev             20480  2
exfat                  73728  0
```

This confirms that the `exfat` kernel module was successfully built and loaded in Dom0.

### Note

In this setup, reusing the existing `extra-configs.cfg` turned out to be more reliable than introducing a new standalone config fragment via `.bbappend`.

## Result

Xen was successfully initialized on Raspberry Pi 4, and the Dom0 Linux kernel booted.

## Conclusion

Running Xen on Raspberry Pi 4 with Yocto is feasible, but requires careful configuration of layers, boot flow, and device tree.

Running Xen on Raspberry Pi 4 with Yocto is feasible, but requires careful configuration of layers, boot flow, and device tree.
