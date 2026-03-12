---
layout: post
title: Building Yocto for Raspberry Pi 4 on Fedora (OrbStack VM)
date: 2026-03-11
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
```

- **MACHINE**: Target machine configuration.
- **BB_NUMBER_THREADS**: Number of parallel BitBake tasks.
- **PARALLEL_MAKE**: Number of threads used by `make`.
- **rm_work**: Removes temporary build files after packages are built to reduce disk usage.
- **IMAGE_FSTYPES**: Specifies the image format to generate.

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
