---
layout: post
title: "Building an A/B RootFS OTA Update System with Yocto, RAUC, and U-Boot on QEMU ARM64"
date: 2026-11-01
tags: [yocto, rauc, qemu, arm64, a/b boot]
categories: Yocto RAUC
---

## Introduction

Reliable over-the-air (OTA) updates are an important requirement for embedded Linux devices. Unlike desktop systems, many embedded products operate unattended, making it essential to recover automatically when a software update fails.

A common solution is an **A/B RootFS update architecture**, where two independent root filesystems are maintained. The system boots from one partition while an update is installed into the inactive partition. If the updated system fails to boot successfully, the bootloader can fall back to the previous version.

In this project, I built and tested an A/B RootFS OTA update system using:

- Yocto Project 5.0 (Scarthgap)
- QEMU ARM64 (`qemuarm64`)
- U-Boot 2024.01
- RAUC 1.15.2
- Linux kernel 6.6
- GPT partitioning and ext4 root filesystems

The implementation supports signed update bundles, inactive-slot installation, boot attempt tracking, and automatic boot-slot rollback.

Most importantly, the complete rollback sequence was verified on QEMU rather than relying solely on configuration or theoretical behavior.

## 1. System Architecture

The system uses a GPT-partitioned virtual disk with three partitions:

| Partition | Device      | Filesystem | Purpose             |
|-----------|-------------|------------|---------------------|
| Boot      | `/dev/vda1` | FAT        | Shared Linux kernel |
| RootFS A  | `/dev/vda2` | ext4       | System slot A       |
| RootFS B  | `/dev/vda3` | ext4       | System slot B       |

Only the root filesystem is redundant. Both slots share the same Linux kernel stored in the FAT boot partition.

The high-level architecture is:

```text
                 U-Boot
                    |
              Bootchooser
                    |
           BOOT_ORDER = A B
                    |
          +---------+---------+
          |                   |
       Slot A              Slot B
     /dev/vda2           /dev/vda3
       ext4                ext4
          |                   |
          +---------+---------+
                    |
              Shared Kernel
                /dev/vda1
```

The bootloader chooses a root filesystem based on persistent environment variables. The selected root partition is passed to Linux through the kernel command line.

For example:

```text
console=ttyAMA0,115200 root=/dev/vda2 rw rootwait
```

This boots RootFS A.

### Why A/B RootFS?

An A/B architecture provides several advantages:

1. The currently running root filesystem is not overwritten during an update.
2. The previous version remains available as a fallback.
3. Update activation requires only a boot-slot switch.
4. A failed update can be recovered without manually restoring a disk image.

However, this implementation does not yet protect against a failed shared-kernel update. Adding Kernel A/B would be a natural extension.

## 2. Integrating RAUC into Yocto

RAUC is an embedded Linux update framework that supports signed update bundles, slot management, and bootloader integration.

I used a custom Yocto layer, `meta-ab`, alongside `meta-rauc`.

The relevant project structure is:

```text
yocto/
├── poky/
├── meta-rauc/
├── meta-ab/
│   ├── recipes-core/
│   │   ├── images/
│   │   │   └── core-image-full-cmdline.bbappend
│   │   └── rauc/
│   │       ├── rauc-conf.bbappend
│   │       ├── qemuarm64-ab-bundle.bb
│   │       └── files/
│   │           └── ca.cert.pem
│   └── recipes-kernel/
│       └── linux/
│           └── files/
│               └── rauc.cfg
├── build-qemuarm64/
└── ab-test/
    └── keys/
```

The image recipe includes RAUC and U-Boot environment utilities, while the RAUC configuration defines the available root filesystem slots.

### RAUC system configuration

The RAUC configuration is installed at:

```text
/etc/rauc/system.conf
```

Its essential contents are:

```ini
[system]
compatible=qemuarm64-ab
bootloader=uboot

[keyring]
path=/etc/rauc/ca.cert.pem

[slot.rootfs.0]
device=/dev/vda2
type=ext4
bootname=A

[slot.rootfs.1]
device=/dev/vda3
type=ext4
bootname=B
```

The `compatible` value identifies which devices may accept an update bundle.

The `bootname` values map RAUC slots to U-Boot bootloader entries.

The keyring contains the certificate used to verify signed RAUC bundles.

## 3. Configuring U-Boot Bootchooser

The bootloader is responsible for selecting a bootable root filesystem and tracking unsuccessful boot attempts.

I used three main environment variables:

```text
BOOT_ORDER=A B
BOOT_A_LEFT=3
BOOT_B_LEFT=3
```

Their meanings are:

| Variable      | Description                              |
|---------------|------------------------------------------|
| `BOOT_ORDER`  | Preferred order of root filesystem slots |
| `BOOT_A_LEFT` | Remaining boot attempts for Slot A       |
| `BOOT_B_LEFT` | Remaining boot attempts for Slot B       |

The Bootchooser examines `BOOT_ORDER` and selects the first slot whose remaining attempt count is greater than zero.

Before booting a selected slot, U-Boot decrements its attempt counter and saves the environment.

U-Boot script:

```text
bootcmd=run boot_ab
boot_ab=run select_slot; if test -n "${selected_slot}"; then run boot_selected; else echo No bootable slot available; fi
select_slot=setenv selected_slot; for slot in ${BOOT_ORDER}; do if test -z "${selected_slot}"; then run check_slot; fi; done
check_slot=if test "${slot}" = "A"; then run check_a; else if test "${slot}" = "B"; then run check_b; fi; fi
check_a=if test ${BOOT_A_LEFT} -gt 0; then setenv selected_slot A; setenv rootpart 2; fi
check_b=if test ${BOOT_B_LEFT} -gt 0; then setenv selected_slot B; setenv rootpart 3; fi
boot_selected=if run decrement_slot; then run load_kernel; else echo Failed to save boot environment; fi
decrement_slot=if test "${selected_slot}" = "A"; then setexpr BOOT_A_LEFT ${BOOT_A_LEFT} - 1; else setexpr BOOT_B_LEFT ${BOOT_B_LEFT} - 1; fi; saveenv
load_kernel=virtio scan; if fatload virtio 0:1 ${kernel_addr_r} Image; then setenv bootargs "console=ttyAMA0,115200 root=/dev/vda${rootpart} rw rootwait"; booti ${kernel_addr_r} - ${fdt_addr}; fi
```

If Slot A has no remaining attempts, U-Boot skips it and tries Slot B.

The selected slot determines the kernel's `root=` argument:

```text
Slot A -> root=/dev/vda2
Slot B -> root=/dev/vda3
```

### Persistent U-Boot environment

The Linux system accesses the U-Boot environment using `fw_printenv` and `fw_setenv`.

The environment configuration is:

```text
/dev/mtd0 0x04000000 0x00040000 0x00040000
```

This is stored in `/etc/fw_env.config` and corresponds to the QEMU flash layout used in this experiment.

For example:

```bash
fw_printenv BOOT_ORDER BOOT_A_LEFT BOOT_B_LEFT
```

The important requirement is that U-Boot and Linux access the same persistent environment storage.

## 4. Kernel Configuration for RAUC Bundles

During the initial integration, the RAUC bundle was generated and signed successfully, but installation failed with:

```text
Failed mounting bundle:
Failed to open /dev/loop-control
```

The kernel had loop-device support configured as a module, but the required module was not available in the target root filesystem.

The verity bundle format also requires the appropriate filesystem and device-mapper capabilities.

I added the following kernel configuration fragment:

```text
CONFIG_BLK_DEV_LOOP=y
CONFIG_SQUASHFS=y
CONFIG_BLK_DEV_DM=y
CONFIG_DM_VERITY=y
```

Building these features directly into the kernel simplified the QEMU setup.

After rebuilding and deploying the kernel, RAUC could mount and process the update bundle successfully.

This was an important integration lesson: **successfully generating a RAUC bundle does not guarantee that the target kernel has all the capabilities required to install it.**

## 5. Creating a Signed RAUC Bundle

RAUC bundles package the update image together with metadata and a cryptographic signature.

The custom Yocto bundle recipe is:

```bitbake
SUMMARY = "QEMU ARM64 A/B RootFS OTA Bundle"
DESCRIPTION = "Signed RAUC update bundle for QEMU ARM64"

LICENSE = "MIT"

inherit bundle

RAUC_BUNDLE_COMPATIBLE = "qemuarm64-ab"
RAUC_BUNDLE_VERSION = "1.0.2"
RAUC_BUNDLE_DESCRIPTION = "QEMU ARM64 RootFS OTA Update"

RAUC_BUNDLE_FORMAT = "verity"

RAUC_BUNDLE_SLOTS = "rootfs"
RAUC_SLOT_rootfs = "core-image-full-cmdline"
RAUC_SLOT_rootfs[fstype] = "ext4"

RAUC_KEY_FILE = "${TOPDIR}/../ab-test/keys/rauc.key.pem"
RAUC_CERT_FILE = "${TOPDIR}/../ab-test/keys/rauc.cert.pem"
```

The private signing key is kept outside the Yocto layer, while the public certificate is installed in the target image.

For this experiment, I used a self-signed development certificate. A production implementation should use a properly managed signing infrastructure and protect the private key.

The private signing key was created by:

```bash
openssl req -x509 -newkey rsa:4096 \
  -keyout ~/yocto/ab-test/keys/rauc.key.pem \
  -out ~/yocto/ab-test/keys/rauc.cert.pem \
  -days 3650 -nodes \
  -sha256 \
  -subj "/CN=QEMU ARM64 RAUC Development/"
```

Then put the key into Yocto layer:

```bash
cp ~/yocto/ab-test/keys/rauc.cert.pem \
  ~/yocto/meta-ab/recipes-core/rauc/files/ca.cert.pem
```

Build the bundle with:

```bash
bitbake qemuarm64-ab-bundle
```

The resulting bundle is deployed under:

```text
tmp/deploy/images/qemuarm64/
```

### Verifying the bundle

Before transferring the bundle to QEMU, I verified its signature and metadata:

```bash
rauc \
  --keyring=../meta-ab/recipes-core/rauc/files/ca.cert.pem \
  info \
  tmp/deploy/images/qemuarm64/qemuarm64-ab-bundle-qemuarm64.raucb
```

The verification reported:

```text
Verified inline signature by
'CN = QEMU ARM64 RAUC Development'

Compatible: 'qemuarm64-ab'
Version: '1.0.2'
Bundle Format: verity
```

This confirmed that the bundle signature was valid and its compatibility identifier matched the target system.

## 6. First OTA Update: Installing Version 1.0.1 Successfully

Before testing rollback behavior, I first verified that a normal OTA update could be installed, activated, and confirmed successfully.

The objective was to update the system from **RootFS A to RootFS B**, boot the updated system, and mark the new slot as good.

### 6.1 Preparing the Version 1.0.1 RootFS

To make the running software version easy to identify, I added a version file to the root filesystem using a Yocto image post-processing function.

In `meta-ab/recipes-core/images/core-image-full-cmdline.bbappend`:

```bitbake
ROOTFS_POSTPROCESS_COMMAND += "write_ab_version; "

write_ab_version() {
    echo "1.0.1" > ${IMAGE_ROOTFS}/etc/ab-version
}
```

This creates the following file in the generated root filesystem:

```text
/etc/ab-version
```

Its contents are:

```text
1.0.1
```

This provides a simple way to verify which software version is actually running after reboot.

### 6.2 Building the Signed Update Bundle

For the first OTA update, I configured the RAUC bundle version as:

```bitbake
RAUC_BUNDLE_VERSION = "1.0.1"
```

Then built the bundle:

```bash
cd ~/yocto/build-qemuarm64

bitbake qemuarm64-ab-bundle
```

The resulting bundle was generated in:

```text
tmp/deploy/images/qemuarm64/
└── qemuarm64-ab-bundle-qemuarm64.raucb
```

Before installation, I verified the bundle signature:

```bash
rauc \
  --keyring=../meta-ab/recipes-core/rauc/files/ca.cert.pem \
  info \
  tmp/deploy/images/qemuarm64/qemuarm64-ab-bundle-qemuarm64.raucb
```

The verification confirmed that the bundle was signed and compatible with the target system.

```text
Compatible: 'qemuarm64-ab'
Version: '1.0.1'
Bundle Format: verity
```

### 6.3 Transferring the Bundle to QEMU

The QEMU guest was configured with user-mode networking and SSH port forwarding:

```bash
-netdev user,id=net0,hostfwd=tcp::2222-:22
-device virtio-net-pci,netdev=net0
```

This forwarded TCP port 2222 on the host to SSH port 22 in the guest.

After starting the SSH server in the guest, I transferred the bundle from the Yocto build host:

```bash
scp -P 2222 \
  tmp/deploy/images/qemuarm64/qemuarm64-ab-bundle-qemuarm64.raucb \
  root@127.0.0.1:/var/tmp/update.raucb
```

The bundle was now available inside the guest at:

```text
/var/tmp/update.raucb
```

### 6.4 Installing the Update into the Inactive Slot

The system was initially running from RootFS A (`/dev/vda2`).

I installed the bundle using:

```bash
rauc install /var/tmp/update.raucb
```

RAUC verified the bundle and installed the new root filesystem into the inactive slot, RootFS B (`/dev/vda3`).

The active RootFS A was not overwritten.

After successful installation, RAUC activated Slot B as the next boot target.

At this point, the system was still running from Slot A. The updated RootFS B would become active only after reboot.

### 6.5 Rebooting into Version 1.0.1

After installation, I rebooted the QEMU guest:

```bash
reboot
```

The U-Boot Bootchooser selected Slot B and passed the following root filesystem argument to Linux:

```text
root=/dev/vda3
```

Inside the guest, I verified the kernel command line:

```bash
cat /proc/cmdline
```

Output:

```text
console=ttyAMA0,115200 root=/dev/vda3 rw rootwait
```

I also checked the installed software version:

```bash
cat /etc/ab-version
```

Output:

```text
1.0.1
```

These results confirmed that the system was running the updated RootFS B rather than the original RootFS A.

### 6.6 Confirming the New Slot as Good

The image included a SysV init startup script for RAUC boot confirmation:

```text
/etc/init.d/rauc-mark-good
```

It was enabled through the startup links:

```text
/etc/rc2.d/S99rauc-mark-good
/etc/rc5.d/S99rauc-mark-good
```

The script invoked the RAUC boot confirmation command:

```bash
rauc status mark-good
```

This informed the bootloader backend that the current slot had booted successfully.

In this implementation, successful confirmation restored the selected slot's boot attempt counter to 3.

I checked the resulting U-Boot environment:

```bash
fw_printenv BOOT_ORDER BOOT_A_LEFT BOOT_B_LEFT
```

Output:

```text
BOOT_ORDER=B A
BOOT_A_LEFT=3
BOOT_B_LEFT=3
```

The boot order now preferred Slot B, and its remaining boot attempts had been restored.

### 6.7 Verifying RAUC Slot Status

Finally, I checked the RAUC system status:

```bash
rauc status
```

The output showed:

```text
=== System Info ===
Compatible:  qemuarm64-ab
Booted from: rootfs.1 (/dev/vda3)

=== Bootloader ===
Activated: rootfs.1 (B)
```

Both slots were reported as good, and Slot B was the currently booted and activated slot.

The results confirmed that the first OTA update had completed successfully.

| Verification          | Expected    | Actual      |
|-----------------------|-------------|-------------|
| Target slot           | B           | B           |
| Root partition        | `/dev/vda3` | `/dev/vda3` |
| Software version      | 1.0.1       | 1.0.1       |
| Bootloader activation | B           | B           |
| Boot confirmation     | Successful  | Successful  |
| `BOOT_B_LEFT`         | 3           | 3           |
| Previous RootFS A     | Preserved   | Preserved   |

**The normal OTA update path was now fully verified:**

```text
Running RootFS A
       |
       v
Install signed RAUC bundle v1.0.1
       |
       v
Write inactive RootFS B
       |
       v
Activate Slot B
       |
       v
Reboot into RootFS B
       |
       v
Verify version 1.0.1
       |
       v
RAUC mark-good
       |
       v
BOOT_B_LEFT restored to 3
       |
       v
OTA Update Successful
```

This established version 1.0.1 on Slot B as the stable baseline for the subsequent rollback experiment.

The next test installed version 1.0.2 into Slot A, deliberately disabled automatic boot confirmation, and verified that U-Boot would eventually return to the stable version 1.0.1 on Slot B.

## 7. Preparing a Rollback Test

To test the rollback mechanism, I built a second bundle containing RootFS version 1.0.2.

The test deliberately prevented the new version from automatically confirming a successful boot.

### Boot confirmation with RAUC

After a successful boot, RAUC can mark the current slot as good:

```bash
rauc status mark-good
```

With this U-Boot Bootchooser implementation, marking a slot as good restores its remaining boot attempts.

The original image included a SysV init script that automatically invoked `mark-good` during startup.

For the rollback test, I disabled its startup links in version 1.0.2.

The image recipe used a post-processing function:

```bitbake
ROOTFS_POSTPROCESS_COMMAND += "write_ab_version; disable_auto_mark_good; "

write_ab_version() {
    echo "1.0.2" > ${IMAGE_ROOTFS}/etc/ab-version
}

disable_auto_mark_good() {
    rm -f ${IMAGE_ROOTFS}/etc/rc2.d/S99rauc-mark-good
    rm -f ${IMAGE_ROOTFS}/etc/rc5.d/S99rauc-mark-good
}
```

This ensured that the new system could boot normally without automatically resetting its boot attempt counter.

It simulated a system that started but never received a successful health confirmation.

This was a controlled test condition, not an actual application failure.

## 8. Installing Version 1.0.2 into Slot A

At the beginning of the rollback test, the system was running the stable version 1.0.1 from Slot B.

I transferred the signed bundle into the QEMU guest using SCP:

```bash
scp -P 2222 \
  tmp/deploy/images/qemuarm64/qemuarm64-ab-bundle-qemuarm64.raucb \
  root@127.0.0.1:/var/tmp/update-1.0.2.raucb
```

Then installed it:

```bash
rauc install /var/tmp/update-1.0.2.raucb
```

RAUC wrote the new root filesystem to the inactive Slot A and activated it for the next boot.

Before rebooting, the system reported:

```text
Booted from: rootfs.1 (/dev/vda3)
Activated: rootfs.0 (A)
```

The U-Boot environment was:

```text
BOOT_ORDER=A B
BOOT_A_LEFT=3
BOOT_B_LEFT=3
```

This confirmed that the currently running system remained on Slot B, while the next boot would attempt Slot A.

## 9. Testing Automatic Rollback

This was the most important part of the experiment.

I rebooted the system repeatedly without calling `rauc status mark-good`.

The bootloader decremented Slot A's attempt counter before each boot.

### First boot: Slot A, version 1.0.2

After the first reboot:

```text
root=/dev/vda2
```

```text
1.0.2
```

U-Boot environment:

```text
BOOT_ORDER=A B
BOOT_A_LEFT=2
BOOT_B_LEFT=3
```

The new root filesystem booted successfully, but its boot attempt counter was not restored.

### Second boot: Slot A, version 1.0.2

After the second reboot:

```text
BOOT_ORDER=A B
BOOT_A_LEFT=1
BOOT_B_LEFT=3
```

Slot A was still selected, and the counter continued decreasing.

### Third boot: Slot A, version 1.0.2

After the third reboot:

```text
BOOT_ORDER=A B
BOOT_A_LEFT=0
BOOT_B_LEFT=3
```

The system was still running version 1.0.2 from Slot A.

However, RAUC now reported:

```text
Booted from: rootfs.0 (/dev/vda2)
Activated: rootfs.1 (B)
```

Slot states:

```text
Slot A: booted, bad
Slot B: inactive, good
```

This is an interesting distinction.

`Booted from` identifies the root filesystem currently running Linux.

`Activated` reflects the bootloader's next selectable boot target.

Even though Linux was still running from Slot A, RAUC recognized that Slot A had exhausted its remaining attempts and that Slot B was the next viable boot target.

### Fourth boot: Automatic fallback to Slot B

After the fourth reboot, the kernel command line showed:

```text
console=ttyAMA0,115200 root=/dev/vda3 rw rootwait
```

The version file reported:

```text
1.0.1
```

RAUC reported:

```text
Booted from: rootfs.1 (/dev/vda3)
Activated: rootfs.1 (B)
```

Slot states:

```text
Slot B: booted, good
Slot A: inactive, bad
```

And the U-Boot environment was:

```text
BOOT_ORDER=A B
BOOT_A_LEFT=0
BOOT_B_LEFT=3
```

**The rollback succeeded.**

The system automatically selected the previous stable RootFS even though `BOOT_ORDER` still listed Slot A first.

The Bootchooser correctly skipped Slot A because its remaining boot attempts had reached zero.

### Complete test results

| Boot    | RootFS | Version | A attempts left | Result              |
|---------|--------|---------|-----------------|---------------------|
| Initial | B      | 1.0.1   | 3               | Stable              |
| First   | A      | 1.0.2   | 2               | New version booted  |
| Second  | A      | 1.0.2   | 1               | No confirmation     |
| Third   | A      | 1.0.2   | 0               | Slot A marked bad   |
| Fourth  | B      | 1.0.1   | 0               | Rollback successful |

The repeated reboots were manually triggered. The boot-slot selection and fallback were automatic.

## 10. Lessons Learned

Several important engineering lessons emerged from this experiment.

### 10.1 Boot success and system health are different

A Linux system reaching userspace does not necessarily mean the product is healthy.

A production system may need to verify:

- Critical application processes
- Hardware initialization
- Network connectivity
- Persistent storage availability
- Application-specific readiness conditions

Only after these checks pass should the system call:

```bash
rauc status mark-good
```

Automatically calling `mark-good` at the beginning of the init sequence is convenient for basic integration testing but provides limited protection against application-level failures.

### 10.2 Boot attempt counters require persistent storage

The remaining boot attempts must survive a reboot or unexpected reset.

In this project, U-Boot saved its environment to persistent QEMU flash storage.

Without persistent boot state, the bootloader could not reliably track repeated unsuccessful attempts.

### 10.3 A slot can be running and marked bad simultaneously

During the third boot, Slot A was still running Linux, but its remaining boot attempts had reached zero.

RAUC correctly reported:

```text
Booted from: A
Activated: B
```

This distinction is useful when debugging update and recovery behavior.

### 10.4 Rollback depends on the bootloader

RAUC manages bundle verification, installation, slot metadata, and bootloader integration.

However, in this architecture, U-Boot performs the actual boot-slot selection.

RAUC does not need to copy the previous root filesystem back over the updated one.

The previous version remains intact in its own partition.

### 10.5 Shared-kernel designs have limitations

The current design provides RootFS A/B redundancy, but both root filesystems depend on the same kernel image.

A corrupted or incompatible shared kernel could prevent either slot from booting.

For a more complete production design, both kernel and root filesystem should be considered in the update and rollback strategy.

## 11. Limitations and Future Work

The current implementation successfully demonstrates signed RAUC updates, A/B RootFS switching, boot confirmation, and automatic boot-slot rollback through U-Boot Bootchooser.

Although the core OTA workflow has been validated, several limitations and potential improvements remain.

### 11.1 Adaptive Updates (Delta OTA)

The current implementation uses full-image RAUC bundles. Each update transfers and installs a complete ext4 root filesystem image, even when only a small portion of the filesystem has changed.

RAUC supports **Adaptive Updates** through its `block-hash-index` mechanism, which can reuse matching data blocks already present on the device and reduce the amount of data downloaded over the network.

Unlike traditional binary delta approaches, this mechanism does not require generating a separate patch for each pair of software versions.

The existing `verity` bundle format provides a suitable foundation for enabling Adaptive Updates.

A future experiment could:

- Enable `block-hash-index` in the RAUC bundle configuration.
- Configure persistent storage for adaptive update metadata.
- Serve update bundles through an HTTP server supporting Range Requests.
- Compare network traffic and installation time between full-image and adaptive updates.

This would extend the project from basic A/B OTA functionality to bandwidth-efficient software distribution, which is particularly valuable for embedded devices with limited or costly network connectivity.

### 11.2 Shared Kernel

Both root filesystem slots currently share the same Linux kernel stored in the FAT boot partition.

This simplifies the boot process but introduces a single point of failure. A corrupted or incompatible kernel update could prevent both root filesystem slots from booting.

A possible improvement would be to maintain separate kernel images for Slot A and Slot B, allowing kernel and root filesystem updates to be activated and rolled back together.

### 11.3 Boot Confirmation

The current implementation uses a SysV init script to execute:

```bash
rauc status mark-good
```

This confirms that the system has reached the configured startup stage, but it does not guarantee that all application services are functioning correctly.

During the version 1.0.2 rollback experiment, automatic boot confirmation was deliberately disabled to simulate an unconfirmed update.

For production systems, boot confirmation could be tied to application-specific health checks rather than performed unconditionally during startup.

### 11.4 Reboot and Recovery Scope

In the rollback experiment, system reboots were triggered manually.

U-Boot automatically decremented the remaining boot attempts, detected when Slot A had exhausted its attempts, and selected the previous stable Slot B.

Therefore, **boot-slot rollback was automatic, while reboot triggering was manual**.

Automatic reset mechanisms are outside the scope of this experiment.

### 11.5 Automated OTA Testing

The OTA installation and rollback procedures were verified interactively on QEMU ARM64.

A lightweight automated test script could make the experiment easier to reproduce and help detect regressions when upgrading Yocto, RAUC, U-Boot, or the Linux kernel.

Such a test could verify successful OTA installation, boot confirmation, boot attempt exhaustion, and fallback to the previous root filesystem.

### Summary

The current project has achieved its primary objective: implementing and validating a reliable A/B RootFS OTA update and rollback mechanism using Yocto, RAUC, and U-Boot on QEMU ARM64.

The most interesting next step is to explore **RAUC Adaptive Updates**, extending the existing system to reduce OTA download traffic while preserving the verified A/B rollback architecture.

## Conclusion

This project demonstrates a working A/B RootFS OTA update system built with Yocto, RAUC, U-Boot, and QEMU ARM64.

The experiment successfully verified the complete sequence:

**Signed bundle installation → inactive RootFS update → new slot activation → repeated unconfirmed boots → boot attempt exhaustion → automatic fallback to the previous version.**

The final system returned from version 1.0.2 on Slot A to version 1.0.1 on Slot B without manually modifying the bootloader environment or restoring a disk image.

Although watchdog-triggered resets, application health checks, and Kernel A/B are not yet implemented, the core update and rollback mechanisms have been demonstrated and validated.

For embedded Linux development, QEMU provides an effective environment for experimenting with OTA architecture and recovery logic before deploying to physical hardware.

---

*Technologies: Yocto Project · Embedded Linux · RAUC · U-Boot · QEMU ARM64 · OTA Updates · A/B RootFS · Bootloader Recovery*
