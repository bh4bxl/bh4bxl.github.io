---
layout: post
title: Building Yocto for Raspberry Pi 4 on Fedora (OrbStack VM)
date: 2026-03-12
tag: [yocto, raspberrypi]
categories: [Yocto]
---

## Introduction

With the build environment prepared in the previous post, we can now download the Yocto source code and try to build a minimal image.

## Download Yocto Source

Clone the Yocto Project reference distribution (poky) from the official repository.

```bash
cd /mnt/yocto
mkdir rpi-build && cd rpi-build
git clone -b scarthgap git://git.yoctoproject.org/poky
```

The scarthgap branch corresponds to Yocto Project 5.0.

Clone additional layers required for Raspberry Pi support:

```bash
git clone -b scarthgap git://git.yoctoproject.org/meta-raspberrypi
git clone -b scarthgap git://git.openembedded.org/meta-openembedded
```

- **meta-raspberrypi** provides BSP support for Raspberry Pi boards.
- **meta-openembedded** provides additional packages and recipes.

## Initialize the Build Environment

Initialize the build environment first:

```bash
cd poky
source oe-init-build-env ../build-rpi
```

This script creates the build directory and sets up the environment variables required for the build system.

## Configure the Build

Edit `conf/local.conf` and add or modify the following settings:

```
MACHINE = "raspberrypi4-64"

BB_NUMBER_THREADS = "12"
PARALLEL_MAKE = "-j 12"

INHERIT += "rm_work"

LICENSE_FLAGS_ACCEPTED = "synaptics-killswitch"

IMAGE_FSTYPES = "wic.bz2"

ENABLE_UART = "1"

CMDLINE:append = " video=HDMI-A-1:1920x1080@60"
```

- **MACHINE**: Target machine configuration.
- **BB_NUMBER_THREADS**: Number of parallel BitBake tasks.
- **PARALLEL_MAKE**: Number of threads used by `make`.
- **rm_work**: Removes temporary build files after packages are built to reduce disk usage.
- **IMAGE_FSTYPES**: Specifies the image format to generate.
- **ENABLE_UART = "1"**: Enables the UART interface on Raspberry Pi. This is useful for accessing the serial console during boot.
- **CMDLINE:append**: Appends additional parameters to the Linux kernel command line. `video=HDMI-A-1:1920x1080@60` forces the HDMI output to use a 1920×1080 resolution at 60 Hz

## Add Required Layers

Add the required layers to the build configuration:

```bash
bitbake-layers add-layer ../meta-openembedded/meta-oe
bitbake-layers add-layer ../meta-openembedded/meta-python
bitbake-layers add-layer ../meta-openembedded/meta-networking
bitbake-layers add-layer ../meta-openembedded/meta-multimedia
bitbake-layers add-layer ../meta-raspberrypi
```

After adding the layers, verify the configuration:

```bash
bitbake-layers show-layers
```

The meta-raspberrypi layer provides BSP support for Raspberry Pi boards, while meta-openembedded provides additional packages and recipes used by many images.

## Build the Image

After running `source oe-init-build-env`, the Yocto build environment is initialized and BitBake is ready to use.

The script prints some common build targets:

```
...
You can now run 'bitbake <target>'

Common targets are:
    core-image-minimal
    core-image-full-cmdline
    core-image-sato
    core-image-weston
    meta-toolchain
    meta-ide-support
...
```

To verify that the build environment is working correctly, build the minimal image:

```bash
bitbake core-image-minimal
```

Building `core-image-minimal`  takes approximately half an hour on my system.

```
bitbake core-image-minimal  10.56s user 1.31s system 0% cpu 31:30.03 total
```

## Build Output

After the build finishes, the generated images can be found in:

```
tmp/deploy/images/raspberrypi4-64/
```

The directory contains several output files, for example:

- **`core-image-minimal-raspberrypi4-64.rootfs-*.wic.bz2`**: SD Card Image
- **`Image`**: Kernel
- **`bcm2711-rpi-4-b.dtb`**: Device Tree

The `.wic.bz2` image can be written directly to an SD card for booting on a Raspberry Pi 4.

## Verify the Image

### Running the Image on Raspberry Pi 4

After the build finishes, the generated image can be written directly to an SD card and booted on a Raspberry Pi 4.

Decompress the image, and write it to the SD card:

```bash
bunzip2 core-image-minimal-raspberrypi4-64.rootfs-*.wic.bz2
sudo dd if=core-image-minimal-raspberrypi4-64.rootfs-.wic of=/dev/sdX bs=4M status=progress
sync
```

Insert the SD card into the Raspberry Pi 4 and power it on.

Default login:

```
user: root
password: (empty)
```

### Running the Image with QEMU

In QEMU, we can choose the `raspi4b` machine.

First, resize the image to 256M

```bash
qemu-img resize core-image-minimal-raspberrypi4-64.rootfs-*.wic 256M
```

Then boot the system with QEMU:

```bash
qemu-system-aarch64 \
    -M raspi4b -m 2G -smp 4 \
    -kernel Image -dtb bcm2711-rpi-4-b.dtb \
    -append "earlycon=pl011,0xfe201000 console=tty1 console=ttyAMA1,115200 root=/dev/mmcblk1p2 rootwait rw dwc_otg.fiq_fsm_enable=0 dwc_otg.lpm_enable=0" \
    -drive file=core-image-minimal-raspberrypi4-64.rootfs-*.wic,format=raw,if=sd \
    -serial stdio -monitor telnet:127.0.0.1:5555,server,nowait
```

After the system boots, the display appears in a new window.

However, because the `raspi4b` machine in QEMU does not provide a PCI bus, the USB controller is not available. As a result, USB devices such as keyboards cannot be used, and it is not possible to input commands on `tty1`.

To control the QEMU instance, open another terminal and connect to the monitor interface:

```bash
telnet localhost 5555
```

- **`q`**: exit the QEMU instance
- **`info cpus`**: show CPU information
- **`info registers`**: display CPU register values (useful to check whether the system is still running)
- **`info pci`**: show PCI devices (empty for `raspi4b`)

The image boots successfully on real Raspberry Pi 4 hardware after being written to an SD card.

On QEMU, the `raspi4b` machine is sufficient for early boot validation. The kernel starts correctly, the root filesystem is mounted, and init begins to run. However, in my setup I was not able to obtain an interactive login prompt on `-serial stdio`, even after enabling the serial console and verifying that a getty entry for `ttyAMA1` was generated in the image.

In practice, this means QEMU is still useful for early boot experiments, while real hardware remains the most reliable way to validate the full system. For interactive userspace debugging, using QEMU with the generic `virt` machine may be a more practical option.

### Inspect the Root Filesystem

Mount the image file:

```bash
sudo losetup --show -Pf /path/to/core-image-minimal-raspberrypi4-64.rootfs-*.wic
```

The `--show -Pf` option automatically creates loop devices for each partition.


Check the loop device associated with the image, and mount the root filesystem:

```bash
losetup -a
sudo mount /dev/loopXp2 /mnt/rootfs
```

After mounting, the root filesystem can be inspected under `/mnt/rootfs`.

Unmount the filesystem when finished:

```bash
sudo umount /mnt/rootfs
sudo losetup -d /dev/loopX
```

The root filesystem is usually located in the second partition of the .wic image.
