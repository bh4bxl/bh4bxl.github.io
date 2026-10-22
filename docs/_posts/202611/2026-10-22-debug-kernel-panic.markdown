---
layout: post
title: Debugging an ARM64 Linux Kernel Panic with Yocto, Kdump, and Crash
date: 2026-11-22
tag: [linux, yocto, crash]
categories: Linux
---

## Introduction

Kernel crashes can be difficult to investigate, especially on embedded Linux systems where interactive debugging may not be available after a panic.

Linux provides a mechanism called **kdump**, which uses `kexec` to boot a secondary kernel after a crash. This crash kernel can access the memory of the original kernel through `/proc/vmcore`, allowing developers to preserve and analyze the system state.

In this experiment, I used Yocto Project, QEMU ARM64, kdump, and the `crash` utility to capture and analyze a deliberately triggered Linux kernel panic.

The goal was to build a reproducible debugging environment without requiring physical ARM64 hardware.

## 1. Test Environment

| Component | Configuration |
|---|---|
| Host OS | Fedora Linux (x86_64) |
| Build system | Yocto Project 5.0 (Scarthgap) |
| Target machine | `qemuarm64` |
| Kernel | Linux 6.6.151-yocto-standard |
| Virtual platform | QEMU ARM64 `virt` |
| CPU | 4 × Cortex-A57 |
| Guest RAM | 2 GiB |
| Crash kernel reservation | 256 MiB |
| Dump format | ELF64 AArch64 Core |
| Analysis tool | Crash Utility 9.0.1 |

The host was an x86_64 Linux system, while the target kernel and crash dump were ARM64.

This architecture difference became important during the analysis stage.

## 2. Preparing the Yocto Environment

The Yocto build directory was configured for the ARM64 QEMU machine:

```text
MACHINE = "qemuarm64"
```

Added another configurations:

```text
EXTRA_IMAGE_FEATURES:append = " dbg-pkgs"

IMAGE_INSTALL:append = " kexec-tools makedumpfile"

QB_MEM = "-m 2048"

BB_NUMBER_THREADS = "12"
PARALLEL_MAKE = "-j12"
```

The kernel needed support for kexec and crash dumping, including:

```text
CONFIG_KEXEC=y
CONFIG_KEXEC_CORE=y
CONFIG_KALLSYMS_ALL=y
CONFIG_CRASH_DUMP=y
CONFIG_PROC_VMCORE=y
CONFIG_MAGIC_SYSRQ=y
CONFIG_DEBUG_INFO=y
```

The exact kernel configuration should be verified against the generated `.config`, as some options vary between kernel versions and architectures.

The root filesystem also needed the `kexec` userspace utility.

After building the image, the virtual machine was started with:

```bash
runqemu qemuarm64 nographic bootparams="crashkernel=256M"
```

The kernel was booted with a crash memory reservation:

```text
crashkernel=256M
```

The reservation can be checked in the running guest:

```bash
cat /proc/cmdline
cat /proc/iomem | grep Crash
```

The `cmdline` showed:

```text
root=/dev/vda rw  mem=2048M ip=192.168.7.2::192.168.7.1:255.255.255.0::eth0:off:8.8.8.8 net.ifnames=0 console=ttyAMA0 console=hvc0 swiotlb=0 crashkernel=256M
```

And the `iomem` showed:

```text
  ad800000-bd7fffff : Crash kernel
```

The reserved memory is used to load the crash kernel and provide a protected execution environment after a panic.

## 3. Loading the Crash Kernel

The next step was to preload a secondary kernel using `kexec`.

For this experiment, I reused the same ARM64 kernel image:

```bash
kexec -p /boot/Image \
  --append="root=/dev/vda ro rootfstype=ext4 console=ttyAMA0 nr_cpus=1 reset_devices init=/bin/sh"
```

Important parameters:

- `-p` loads the kernel for the panic path rather than immediately executing it.
- `nr_cpus=1` limits the crash kernel to one CPU.
- `reset_devices` requests device reset behavior in drivers that support it.
- `init=/bin/sh` starts a minimal shell instead of the normal init system.

Using `/bin/sh` was particularly useful because the root filesystem included a kdump init script that automatically attempted to save the dump and reboot.

Bypassing the normal init system allowed me to manually inspect `/proc/vmcore` and transfer it to the host.

## 4. Triggering a Kernel Panic

After loading the crash kernel, I deliberately triggered a panic:

```bash
echo c > /proc/sysrq-trigger
```

This invokes the Magic SysRq crash handler.

The original kernel entered its panic path, and kdump switched execution to the previously loaded crash kernel.

The crash kernel booted with a single CPU and access to the crashed kernel's memory.

I verified that `/proc/vmcore` was available and contained an ELF header:

```bash
od -An -tx1 -N16 /proc/vmcore
```

The first four bytes were:

```text
7f 45 4c 46
```

These are the ELF magic bytes.

## 5. Transferring vmcore to the Fedora Host

The guest's root filesystem was relatively small, and the raw crash dump was approximately 1.8 GiB.

Instead of saving the dump inside the guest, I transferred it directly to the Fedora host over the QEMU TAP network.

The network configuration was:

```text
Fedora host: 192.168.7.1
QEMU guest:  192.168.7.2
```

On the Fedora host, I started a TCP listener using Netcat:

```bash
nc -l 192.168.7.1 9999 > ~/vmcore
```

In the crash kernel environment, I configured the network interface:

```bash
ip link set eth0 up
ip addr add 192.168.7.2/24 dev eth0
ping -c 3 192.168.7.1
```

If the address was already assigned through kernel boot parameters, it was not necessary to add it again.

I then streamed the raw dump to the host:

```bash
nc 192.168.7.1 9999 < /proc/vmcore
```

The resulting dump was:

```text
1,904,488,448 bytes
```

At approximately 7.2 MB/s, the transfer took around four to five minutes.

This approach avoided installing or starting SSH in the minimal crash kernel environment and did not require additional storage inside the guest.

## 6. Validating the ELF Core Dump

On Fedora, I inspected the captured file:

```bash
readelf -h ~/vmcore
readelf -lW ~/vmcore
```

The ELF header showed:

```text
Class:   ELF64
Type:    CORE
Machine: AArch64
```

The dump contained one `NOTE` segment and three `LOAD` segments.

The final load segment was:

```text
Offset:  0x6f043000
FileSiz: 0x02800000
```

Its ending offset was:

```text
0x6f043000 + 0x02800000 = 0x71843000
```

This matched the actual file length:

```text
1,904,488,448 bytes
```

The agreement provided a useful structural check that the transfer had not been truncated. It did not constitute a full checksum or semantic integrity verification.

## 7. Preparing vmlinux with Debug Symbols

The `crash` utility requires a kernel image that matches the crashed kernel and contains the necessary debugging information.

Yocto retained an uncompressed `vmlinux` in the kernel build directory:

```text
tmp/work/qemuarm64-poky-linux/linux-yocto/6.6.151+git/linux-qemuarm64-standard-build/vmlinux
```

I verified that it contained DWARF debug information and a symbol table:

```bash
readelf -S vmlinux | grep -E '\.debug_info|\.symtab'
```

The output included:

```text
.debug_info
.symtab
```

This made the file suitable for symbol and kernel structure analysis, provided that it matched the exact kernel build that produced the dump.

## 8. Building Crash for ARM64 Analysis

The first attempt to analyze the dump using Fedora's packaged `crash` utility failed:

```text
WARNING: machine type mismatch:
         crash utility: X86_64
         vmlinux: ARM64

WARNING: machine type mismatch:
         crash utility: X86_64
         vmcore: ARM64
```

Although the Fedora host was x86_64, the crash dump was ARM64.

The solution was to build the crash utility with ARM64 target support:

```bash
git clone https://github.com/crash-utility/crash.git
cd crash

make target=ARM64 -j8
```

The resulting executable still ran on the x86_64 host but understood ARM64 kernel structures and crash dumps.

I then launched the debugger:

```bash
./crash /path/to/vmlinux ~/vmcore
```

The utility successfully loaded the dump.

## 9. Analyzing the Kernel Panic

Inside the crash utility, I started with:

```text
crash> bt
```

The backtrace identified the crashing task:

```text
PID: 346
CPU/NUMA: 3/0
COMMAND: "sh"
```

The relevant call chain was:

```text
machine_kexec
__crash_kexec
panic
sysrq_handle_crash
__handle_sysrq
write_sysrq_trigger
proc_reg_write
vfs_write
ksys_write
__arm64_sys_write
invoke_syscall
el0_svc
```

This matched the expected execution path for:

```bash
echo c > /proc/sysrq-trigger
```

The ARM64 register dump also showed:

```text
X8:        0x40
SYSCALLNO: 40
X0:        1
X2:        2
```

On AArch64 Linux, syscall number 64 corresponds to `write()`.

The two-byte write was consistent with the shell writing `c\n` to `/proc/sysrq-trigger`.

The call stack therefore demonstrated the full transition from a userspace system call to the kernel panic handler and finally to the kexec crash path.

## 10. Examining All CPUs During the Crash

One of the most interesting results came from:

```text
crash> bt -a
```

The original kernel was running with four CPUs.

The dump showed:

| CPU | Task | State |
|---|---|---|
| CPU 0 | `swapper/0` | Idle, interrupted by IPI |
| CPU 1 | `swapper/1` | Idle, interrupted by IPI |
| CPU 2 | `swapper/2` | Idle, interrupted by IPI |
| CPU 3 | `sh` | Executing the panic and kexec path |

For CPUs 0–2, the backtrace began with:

```text
crash_save_cpu
ipi_handler
handle_percpu_devid_irq
handle_irq_desc
generic_handle_domain_irq
gic_handle_irq
```

The interrupted execution context pointed into:

```text
default_idle_call
do_idle
cpu_startup_entry
```

This revealed an important detail about SMP crash handling.

CPU 3 triggered the panic and entered `machine_kexec()`, while the other CPUs received inter-processor interrupts and entered the crash-state saving path.

The backtraces also crossed the ARM64 IRQ stack boundary:

```text
--- <IRQ stack> ---
```

This showed that the crash dump preserved enough information to unwind both the interrupt handler stack and the interrupted kernel task stack.

Even though the crash kernel booted with only one CPU, the dump still contained information about all four CPUs from the original kernel.

## 11. Problems Encountered and Lessons Learned

### Automatic kdump scripts can interfere with manual experiments

When I initially booted the crash kernel with the normal init system, a SysVinit kdump script detected `/proc/vmcore`, attempted to save it using `makedumpfile`, and rebooted the system.

This made interactive investigation difficult.

Using:

```text
init=/bin/sh
```

avoided the automatic dump-and-reboot sequence.

### makedumpfile may require additional kernel information

During the experiment, `makedumpfile` reported:

```text
Can't open the kernel file
(/boot/vmlinux-6.6.151-yocto-standard)
```

The configured dump mode required kernel information that was not available at the expected location.

For the initial experiment, transferring the raw `/proc/vmcore` was a simpler solution.

### Minimal init environments require manual setup

Booting directly into `/bin/sh` avoids unnecessary services, but networking and other system facilities may need to be configured manually.

It is also important to use bounded commands such as:

```bash
ping -c 3 192.168.7.1
```

A PID 1 shell without a conventional controlling terminal can make interrupting foreground commands unexpectedly difficult.

### Crash analysis tools are architecture-specific

The host architecture and the crash dump architecture do not need to be the same.

However, the `crash` utility must be built with support for the target kernel architecture.

For this experiment:

```bash
make target=ARM64
```

was essential.

## 12. Conclusion

This experiment demonstrated a complete ARM64 kernel crash debugging workflow using Yocto and QEMU:

```text
Yocto ARM64 Linux
       |
       v
Reserve crashkernel memory
       |
       v
Load crash kernel with kexec -p
       |
       v
Trigger Kernel Panic using SysRq
       |
       v
Boot secondary crash kernel
       |
       v
Access /proc/vmcore
       |
       v
Transfer raw vmcore to Fedora
       |
       v
Analyze with ARM64 Crash Utility
       |
       v
Recover kernel and per-CPU backtraces
```

The most useful result was not simply obtaining a valid vmcore file, but recovering the actual execution state of the original four-CPU ARM64 kernel.

The analysis showed the panic-triggering task, the userspace-to-kernel syscall path, the transition into `machine_kexec()`, and the IPI-based crash handling on the other CPUs.

This provides a practical foundation for investigating real kernel crashes in embedded Linux systems, including driver failures, memory corruption, and synchronization problems.

### Possible Next Steps

- Automate crash dump collection with `makedumpfile`.
- Compare raw and filtered vmcore sizes.
- Analyze a real kernel module crash instead of a deliberate SysRq panic.
- Investigate ARM64 crash shutdown and CPU register saving in the kernel source.
- Integrate crash dump collection into a Yocto-based debugging image.
