---
layout: post
title: "Building an A/B Boot System with Yocto and U-Boot on QEMU ARM64"
date: 2026-10-24
description: "Building and testing an A/B root filesystem boot flow with Yocto Scarthgap, U-Boot, Wic, and QEMU ARM64."
tags: [yocto, u-boot, qemu, arm64, a/b boot]
categories: Yocto U-Boot
---

An A/B root filesystem layout lets an embedded device boot from one system partition while keeping another available for an update or fallback. In this project, I built an A/B boot demonstration using **Yocto Project 5.0 (Scarthgap)**, **U-Boot 2024.01**, and **QEMU ARM64**.

The goal of this first stage was to make the boot flow reproducible through a custom Yocto layer, rather than relying on manually partitioned disk images and interactive U-Boot commands. OTA installation with RAUC and migration to RAUC's standard U-Boot bootchooser will be covered separately.

## Architecture

The QEMU virtual machine uses a GPT-partitioned VirtIO disk and two 64 MiB NOR pflash devices:

```text
QEMU ARM64 (virt)
|
+-- pflash0 (64 MiB): U-Boot firmware
+-- pflash1 (64 MiB): persistent U-Boot environment
|
+-- VirtIO disk: Yocto-generated .wic image (~3.13 GiB)
    +-- /dev/vda1  FAT32  128 MiB   Shared Linux Image
    +-- /dev/vda2  ext4   1536 MiB  RootFS A
    +-- /dev/vda3  ext4   1536 MiB  RootFS B
```

Both slots currently share the same kernel in the FAT boot partition. **This is a root-filesystem A/B demonstration, not a complete atomic kernel-plus-rootfs OTA design.**

### Boot sequence

1. U-Boot loads its environment from NOR flash, or uses the compiled default environment on a fresh flash image.
2. `active_slot` selects partition 2 (A) or 3 (B).
3. U-Boot loads `Image` from the shared FAT partition.
4. U-Boot sets the Linux `root=` parameter and starts the kernel.
5. Linux can inspect or update the persistent U-Boot environment with `fw_printenv` and `fw_setenv`.

## 1. Create a Yocto layer

The project uses a custom layer named `meta-ab`. Its relevant files are organized as follows:

```text
meta-ab
├── conf
│   └── layer.conf
├── recipes-bsp
│   └── u-boot
│       ├── files
│       │   ├── 0001-qemu-arm-add-ab-boot-environment.patch
│       │   └── bootcount.cfg
│       └── u-boot_%.bbappend
├── recipes-core
│   └── images
│       └── core-image-full-cmdline.bbappend
├── recipes-kernel
│   └── linux
│       ├── files
│       │   └── mtd.cfg
│       └── linux-yocto_%.bbappend
├── recipes-support
│   └── ab-env
│       ├── ab-env-config_1.0.bb
│       └── files
│           └── fw_env.config
└── wic
    └── qemuarm64-ab.wks
```

The build uses `MACHINE = "qemuarm64"` and `core-image-full-cmdline`. Add `meta-ab` to `BBLAYERS` before building.

## 2. Enable U-Boot boot counting

In `recipes-bsp/u-boot/files/bootcount.cfg`:

```text
CONFIG_BOOTCOUNT_LIMIT=y
CONFIG_BOOTCOUNT_ENV=y
```

The U-Boot `recipes-bsp/u-boot/u-boot_%.bbappend` adds this fragment and a board environment patch:

```bitbake
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"

SRC_URI:append:qemuarm64 = " \
    file://bootcount.cfg \
    file://0001-qemu-arm-add-ab-boot-environment.patch \
"

do_configure:append:qemuarm64() {
    ${S}/scripts/kconfig/merge_config.sh -m \
        ${B}/.config \
        ${WORKDIR}/bootcount.cfg

    oe_runmake -C ${S} O=${B} olddefconfig
}
```

The patch modifies U-Boot's existing text environment at `board/emulation/qemu-arm/qemu-arm.env`. **Preserve the original QEMU board settings** such as `kernel_addr_r`, `fdt_addr`, and the console configuration, then append the A/B variables.

`recipes-bsp/u-boot/files/0001-qemu-arm-add-ab-boot-environment.patch`:

```diff
From: Local Yocto A/B OTA Project
Subject: [PATCH] qemu-arm: add A/B boot environment
Upstream-Status: Inappropriate [oe-specific]

---

--- a/board/emulation/qemu-arm/qemu-arm.env
+++ b/board/emulation/qemu-arm/qemu-arm.env
@@ -13,3 +13,12 @@
 kernel_addr_r=0x40400000
 ramdisk_addr_r=0x44000000
 boot_targets=qfw usb scsi virtio nvme dhcp
+
+active_slot=A
+bootlimit=3
+upgrade_available=0
+bootcmd=run boot_ab
+altbootcmd=run rollback; run boot_ab
+select_slot=if test "${active_slot}" = "B"; then setenv rootpart 3; else setenv rootpart 2; fi
+boot_ab=run select_slot; virtio scan; fatload virtio 0:1 ${kernel_addr_r} Image; setenv bootargs "console=ttyAMA0,115200 root=/dev/vda${rootpart} rw rootwait"; booti ${kernel_addr_r} - ${fdt_addr}
+rollback=echo Rolling back to Slot A; setenv active_slot A; setenv upgrade_available 0; setenv bootcount 0; saveenv
```

The patch is maintained as part of `meta-ab`, not as a direct modification of the upstream U-Boot checkout. For Yocto patch QA, include an appropriate `Upstream-Status` header in the patch.

### How rollback works

`CONFIG_BOOTCOUNT_ENV` uses `upgrade_available` to control persistent boot counting. Normal boots with `upgrade_available=0` do not repeatedly write a new count to flash. During a trial boot, setting `upgrade_available=1` enables the persistent counter. If the boot count exceeds `bootlimit`, U-Boot invokes `altbootcmd`.

In this initial implementation, rollback **always selects Slot A**. It is intentionally simple: it demonstrates the mechanism, but it is not yet a symmetric production bootchooser.

## 3. Enable NOR flash access in Linux

To let Linux read and modify U-Boot's persistent environment, enable MTD support in the kernel.

`recipes-kernel/linux/files/mtd.cfg`:

```text
CONFIG_MTD=y
CONFIG_MTD_BLOCK=y
CONFIG_MTD_CFI=y
CONFIG_MTD_CFI_INTELEXT=y
CONFIG_MTD_CFI_AMDSTD=y
CONFIG_MTD_PHYSMAP=y
CONFIG_MTD_PHYSMAP_OF=y
CONFIG_MTD_OF_PARTS=y
```

`recipes-kernel/linux/linux-yocto_%.bbappend`:

```bitbake
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"

SRC_URI:append:qemuarm64 = " file://mtd.cfg"
```

On this QEMU setup, Linux reports the two NOR flash banks as one concatenated MTD device:

```text
mtd0: 08000000 00040000 "0.flash"
```

The second 64 MiB bank therefore starts at offset `0x04000000` within `/dev/mtd0`.

## 4. Package fw_env.config into the root filesystem

Install `libubootenv-bin` to provide `fw_printenv` and `fw_setenv`. The matching configuration file must also be installed in the image.

`recipes-support/ab-env/files/fw_env.config`:

```text
# Device       Offset       Env size     Erase size
/dev/mtd0      0x04000000   0x00040000   0x00040000
```

`recipes-support/ab-env/ab-env-config_1.0.bb`:

```bitbake
SUMMARY = "U-Boot environment configuration for QEMU ARM64 A/B"
DESCRIPTION = "Install fw_env.config for accessing U-Boot environment from Linux"
LICENSE = "CLOSED"

SRC_URI = "file://fw_env.config"

S = "${WORKDIR}"

do_install() {
    install -d ${D}${sysconfdir}
    install -m 0644 ${WORKDIR}/fw_env.config ${D}${sysconfdir}/fw_env.config
}

FILES:${PN} = "${sysconfdir}/fw_env.config"
```

Add both packages to the image.

`recipes-core/images/core-image-full-cmdline.bbappend`:

```bitbake
IMAGE_INSTALL:append = " libubootenv-bin ab-env-config"
...
```

**The offset is specific to this QEMU flash topology.** It must be reconsidered if the NOR flash mapping or board changes.

## 5. Generate the A/B disk using Wic

Instead of manually creating GPT partitions, use a Wic kickstart file.

`wic/qemuarm64-ab.wks`:

```text
part /boot --source bootimg-partition --ondisk vda --fstype=vfat --label boot --active --align 1024 --fixed-size 128M
part / --source rootfs --ondisk vda --fstype=ext4 --label rootfs-a --align 1024 --fixed-size 1536M
part / --source rootfs --ondisk vda --fstype=ext4 --label rootfs-b --align 1024 --fixed-size 1536M
bootloader --ptable gpt
```

Configure the image recipe or `.bbappend` to enable Wic, select this `.wks` file, and include the kernel in the FAT partition.

`recipes-core/images/core-image-full-cmdline.bbappend`:

```bitbake
IMAGE_INSTALL:append = " libubootenv-bin ab-env-config"

IMAGE_FSTYPES:append:qemuarm64 = " wic"

WKS_FILE:qemuarm64 = "qemuarm64-ab.wks"

WKS_SEARCH_PATH:prepend = "${LAYERDIR_meta-ab}/wic:"

IMAGE_BOOT_FILES:qemuarm64 = "Image"
```

Ensure the custom `.wks` directory is included in `WKS_SEARCH_PATH` by your layer configuration. Then build:

```bash
cd ~/yocto
source poky/oe-init-build-env build-qemuarm64
bitbake core-image-full-cmdline
```

The resulting `.wic` file was approximately 3.13 GiB. Its GPT partition table was verified as:

```text
Partition  Start sector  Sectors   Size
1          2048          262144    128 MiB
2          264192        3145728   1.5 GiB
3          3409920       3145728   1.5 GiB
```

### Inspect the image without mounting it

The correct `wic ls` syntax in this Yocto version puts the directory path immediately after the partition number:

```bash
IMG=tmp/deploy/images/qemuarm64/core-image-full-cmdline-qemuarm64.rootfs.wic
wic ls "${IMG}:1"
wic ls "${IMG}:2/etc/"
wic ls "${IMG}:3/etc/"
```

Both root filesystems should contain `/etc/fw_env.config`. This was verified for Slot B and subsequently confirmed at runtime for both slots.

## 6. Boot the image in QEMU

This setup requires two 64 MiB pflash images: one containing the built `u-boot.bin`, and another initialized to the erased NOR value `0xFF`. **Use test copies rather than overwriting an existing flash image containing a working environment.**

From `build-qemuarm64`, after preparing those pflash images and making a writable copy of the `.wic` image:

```bash
cp --reflink=auto \
  tmp/deploy/images/qemuarm64/core-image-full-cmdline-qemuarm64.rootfs.wic \
  ../ab-test/ab-wic-test.img
```

Then:

```bash
qemu-system-aarch64 \
  -machine virt -cpu cortex-a57 -smp 4 -m 2048 -nographic \
  -drive if=pflash,format=raw,file=../ab-test/flash0-test.img \
  -drive if=pflash,format=raw,file=../ab-test/flash1-test.img \
  -drive if=none,id=disk0,format=raw,file=../ab-test/ab-wic-test.img \
  -device virtio-blk-pci,drive=disk0
```

With a fresh environment, U-Boot reported:

```text
=> printenv active_slot bootlimit bootcmd altbootcmd kernel_addr_r fdt_addr
active_slot=A
bootlimit=3
bootcmd=run boot_ab
altbootcmd=run rollback; run boot_ab
kernel_addr_r=0x40400000
fdt_addr=0x40000000
```

This confirmed that the patched **compiled default environment** was available without manually entering the boot commands.

## 7. Validate both root filesystem slots

### Slot B

After selecting Slot B, Linux reported:

```console
root@qemuarm64:~# cat /proc/cmdline
console=ttyAMA0,115200 root=/dev/vda3 rw rootwait
root@qemuarm64:~# findmnt /
TARGET
  SOURCE    FSTYPE OPTIONS
/ /dev/vda2 ext4   rw,relatime
root@qemuarm64:~# cat /etc/fw_env.config
# Device       Offset       Env size     Erase size
/dev/mtd0      0x04000000   0x00040000   0x00040000
root@qemuarm64:~# fw_printenv active_slot
active_slot=B
```

### Slot A

After switching back to Slot A, Linux reported:

```console
root@qemuarm64:~# cat /proc/cmdline
console=ttyAMA0,115200 root=/dev/vda2 rw rootwait
root@qemuarm64:~# fw_printenv active_slot
active_slot=A
```

Both root filesystems booted from the Yocto-generated Wic image, and both included the U-Boot environment configuration.

Linux-to-U-Boot environment persistence had also been tested earlier by writing a variable with `fw_setenv`, rebooting, and reading the same value in U-Boot.

## 8. Bootcount and rollback testing

The bootcount mechanism and a simulated failure-triggered rollback to Slot A were verified during the earlier manually assembled image stage. A fresh regression test of the final Wic image and patched U-Boot build is still worth performing before treating this as a release-ready recovery flow.

A controlled trial starts by setting:

```text
active_slot=B
upgrade_available=1
bootcount=0
bootlimit=3
```

The intended behavior is to count trial boots, then run `altbootcmd` after the limit is exceeded. A successful boot must be explicitly confirmed by clearing `upgrade_available` and resetting `bootcount`; the current demonstration does not yet automate that health confirmation.

## 9. What is complete, and what comes next?

**Verified in this project:**

- Custom Yocto layer for U-Boot A/B defaults and bootcount configuration.
- Kernel MTD support for QEMU NOR flash.
- `fw_env.config` installed into the root filesystem by a dedicated recipe.
- Wic-generated GPT disk with a shared boot partition and two ext4 rootfs slots.
- Booting Slot A and Slot B from the generated image.
- Reading U-Boot environment variables from Linux.
- Earlier manual tests of persistent environment updates and failure rollback.

**Still to do:**

- Automate creation of the two QEMU pflash images and a reproducible launch script.
- Re-run the bootcount/rollback regression test against the final Wic-generated setup.
- Integrate RAUC to install signed update bundles into the inactive slot.
- Replace the custom `active_slot`/`bootcount` approach with the **standard RAUC U-Boot bootchooser** (`BOOT_ORDER`, `BOOT_A_LEFT`, `BOOT_B_LEFT`).
- Consider per-slot kernel images so an update can cover both kernel and rootfs.

## Conclusion

The main improvement was moving from a hand-built A/B disk and manually configured U-Boot environment to a **reproducible Yocto build**. U-Boot now carries the A/B boot defaults, the kernel exposes NOR flash to Linux, both rootfs slots include environment access tools, and Wic creates the GPT disk layout automatically.

The next article will extend this baseline with **RAUC and the standard U-Boot bootchooser**, separating the initial boot experiment from a more complete OTA update design.
