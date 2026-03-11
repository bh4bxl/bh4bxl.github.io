---
layout: post
title: Setting Up a Linux Development Environment on macOS (OrbStack + Fedora)
date: 2026-03-11
tag: [fedora, macos, linux kernel]
categories: Linux distribution
---

## Background

Yocto development is typically performed on native Linux systems. However, many developers today use macOS as their primary workstation. In this case, running a Linux virtual machine is often the most practical solution.

Recently I experimented with building a Yocto development environment on macOS using **OrbStack** and **Fedora**. The goal was to create a clean and reproducible environment for building Yocto images and testing them.

## Host Environment

- Host OS: macOS
- Machine: Mac mini, M4 pro (12 CPUs, 8P + 4E)
- Guest OS: Fedora 43

## Installing OrbStack

### Why OrbStack

OrbStack is a fast, lightweight tool for running Docker containers and Linux on macOS. It uses virtualization technology to provide a Linux environment.

### Install OrbStack

It is very easy to install OrbStack using `brew`:

```bash
brew install orbstack
```

### Other Useful Tools

- **asitop**: Perf monitoring CLI tool for Apple Silicon
- **stats**: A macOS system monitor in menu bar

## Installing Fedora

Open OrbStack from the Applications folder and create a Fedora Linux machine using the default parameters.

From the macOS terminal, start and connect to the Fedora machina:

```bash
orb -m fedora
```

Update the system and install the required build dependencies.

### Change Work Dir

After logging into Fedora with `orb`, the default working directory is not the home directory.
To automatically switch to the home directory (`~`), add the following line to `.zshrc`:

```bash
cd ~
```

### Build Linux Kernel

We can verify the environment with building Linux kernel.

To build the Linux kernel, the following packages need to be installed.

```bash
sudo dnf install -y \
    git wget curl diffstat unzip texinfo chrpath socat cpio \
    python3 python3-pip python3-devel python3-jinja2 \
    gcc gcc-c++ make cmake \
    perl perl-Data-Dumper perl-Text-ParseWords perl-Thread-Queue \
    gawk tar bzip2 gzip xz \
    patch diffutils file which \
    ncurses-devel \
    openssl-devel \
    elfutils-libelf-devel \
    bc \
    flex bison \
    rsync \
    ccache \
    zstd
```

Download the Linux kernel source code:

```bash
git clone https://github.com/torvalds/linux.git
git chechout v6.19
```

Build the Linux kernel for arm64 with Clang:

```bash
make ARCH=arm64 LLVM=1 defconfig
make ARCH=arm64 LLVM=1 -j$(nproc)
```

### Fedora disk image location in OrbStack

On macOS, OrbStack stores the virtual machine disk image in the following location: `~/Library/Group\ Containers/HUAQ24HBR6.dev.orbstack/data/data.img.raw":

```bash
ls -lh ~/Library/Group\ Containers/HUAQ24HBR6.dev.orbstack/data/data.img.raw
-rw-r--r-- 1 bh4bxl 8.0T Mar 11 13:37 '/Users/bh4bxl/Library/Group Containers/HUAQ24HBR6.dev.orbstack/data/data.img.raw'
```

The actual size is:

```bash
du -ch ~/Library/Group\ Containers/HUAQ24HBR6.dev.orbstack/data/data.img.raw
20G	/Users/bh4bxl/Library/Group Containers/HUAQ24HBR6.dev.orbstack/data/data.img.raw
20G	total
```

This sparse disk image grows automatically as data is written.

## Using an External Disk for Yocto Builds

Since OrbStack VMs cannot directly attach an additional disk image, we can instead create a disk image file and mount it inside the VM using a loop device.

First, create a sparse image file in Fedora:

```bash
truncate -s 500G /mnt/mac/Volumes/Utils/bh4bxl/yocto_dynamic.img
```

Attach the image file to the next available loop device:

```bash
sudo losetup -f /mnt/mac/Volumes/Utils/bh4bxl/yocto_dynamic.img
```

Then format it and mount it inside Fedora.

```bash
sudo mkfs.ext4 /dev/loop0
sudo mkdir /mnt/yocto
sudo mount /dev/loop0 /mnt/yocto
```

Now the build environment on macOS for Yocto is ready.
