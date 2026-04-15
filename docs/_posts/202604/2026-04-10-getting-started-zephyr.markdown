---
layout: post
title: Getting Started with Zephyr on MCU – A Practical Engineer’s View
date: 2026-04-10
tag: [rtos, pico2, stm32]
categories: RTOS
---

## Background

Recently, I started exploring **Zephyr RTOS** on MCU platforms (Raspberry Pi Pico2w and STM32H7S78-DK).
Coming from a background in **Linux/QNX BSP and driver development**, my initial impression was:

> Zephyr feels familiar in structure, but fundamentally different in execution model.

This blog summarizes my hands-on experience focusing on:

* Build environment setup
* Application structure
* Device model and driver workflow
* Comparison with Linux and bare-metal approaches

---

## Environment Setup (Minimal Practical Flow)

The official setup is:

```bash
mkdir -p ~/zephyrproject
python3 -m venv ~/zephyrproject/.venv
source ~/zephyrproject/.venv/bin/activate
pip install -U pip west
west init ~/zephyrproject
cd ~/zephyrproject
west update
west zephyr-export
west packages pip --install
west sdk install
```

### What actually matters daily

After first-time setup, daily workflow is simply:

```bash
cd ~/zephyrproject
source .venv/bin/activate
```

That’s it.

> From my experience, the Zephyr workflow feels quite similar to the QNX development environment, especially in terms of toolchain setup, build structure, and board-oriented development.

### Updating Zephyr

In day-to-day development, updating Zephyr is straightforward and typically only required when you want to pull newer upstream changes or switch versions.

The standard way to update the workspace is:

```bash
west update
```

This command synchronizes all modules defined in the Zephyr workspace manifest.

In most cases, no additional steps are required. However, when switching to a different Zephyr version or after major updates, it can be helpful to re-export environment variables:

```bash
west zephyr-export
```

Reinstalling Python dependencies is usually unnecessary unless there are changes in requirements:

```bash
west packages pip --install
```

In practice, frequent updates are not required. For stability, it is often better to stick to a specific Zephyr version during development.

---

## Project Structure (Out-of-tree Application)

Unlike Linux kernel development, Zephyr encourages **application-centric development**.

Example (Copy from `~/zephyrproject/zephyr/samples/hello_world`):

```text
hello_world/
├── CMakeLists.txt
├── prj.conf
├── README.rst
├── sample.yaml
└── src
    └── main.c
```

### Build

For Raspberry Pico 2:

```bash
west build -b rpi_pico2/rp2350a/m33 -d ~/projects/zephyr/hello_world/build_pico2 ~/projects/zephyr/hello_world
```

For STM32H7S78-DK, both internal-flash and external-XIP builds are available.

Internal flash build:

```bash
west build -b stm32h7s78_dk -d ~/projects/zephyr/hello_world/build_stm32h7s78_dk ~/projects/zephyr/hello_world
```

External XIP build:

```bash
west build -b stm32h7s78_dk/stm32h7s7xx/ext_flash_app --sysbuild -d ~/projects/zephyr/hello_world/build_stm32h7s_xip ~/projects/zephyr/hello_world
```

The external XIP configuration is intended for running the application from OSPI flash, with MCUboot located in internal flash.

### Workspace and Multi-Board Builds

Zephyr uses a single workspace to support multiple MCU platforms. Unlike some build systems, it is not necessary to create a separate workspace for each board or SoC.

In practice, one workspace (e.g., `~/zephyrproject`) can be used to build applications for different targets:

```bash
west build -b rpi_pico2/rp2350a/m33 -d ~/projects/zephyr/hello_world/build_pico2 ~/projects/zephyr/hello_world
west build -b stm32h7s78_dk -d ~/projects/zephyr/hello_world/build_stm32h7s78_dk ~/projects/zephyr/hello_world
west build -b stm32h7s78_dk/stm32h7s7xx/ext_flash_app --sysbuild -d ~/projects/zephyr/hello_world/build_stm32h7s_xip ~/projects/zephyr/hello_world
```

Each target requires its own build directory, since the build output is tightly coupled with the selected board, including devicetree, configuration, and toolchain settings.

This is conceptually similar to the Yocto build model:

| Yocto                       | Zephyr                    |
| --------------------------- | ------------------------- |
| MACHINE                     | board                     |
| build directory per MACHINE | build directory per board |

In daily development, it is recommended to keep a single workspace and organize builds per board using separate build directories.

### Flashing the Firmware

After building the application, Zephyr provides a unified interface to flash firmware using different runners.

The firmware to be flashed is always taken from the selected build directory:

```bash
west flash -d ~/projects/zephyr/hello_world/build_pico2 --runner probe-rs
west flash -d ~/projects/zephyr/hello_world/build_stm32 --runner openocd
```

The runner only defines how the firmware is programmed (e.g., probe-rs, OpenOCD, J-Link), while the actual binary comes from the corresponding build directory.

For some boards, the default runner can be used directly:

```bash
west flash -d ~/projects/zephyr/hello_world/build_stm32h7s78_dk
```

However, not all boards support all runners. For example, some STM32 boards may not support `probe-rs` in Zephyr, even though the underlying chip is supported. In such cases, alternative runners such as `openocd` or `stm32cubeprogrammer` should be used.

It is also possible to flash firmware directly using external tools such as `probe-rs`:

```bash
probe-rs download --chip <chip_name> build_stm32/zephyr/zephyr.elf
```

In practice, it is recommended to always specify the build directory explicitly to avoid flashing the wrong target when working with multiple boards.

### Debugging

Zephyr also provides a unified interface for debugging through `west debug`, similar to the flashing workflow.

```bash
west debug -d ~/projects/zephyr/hello_world/build_stm32h7s78_dk --runner openocd
```

The debug runner depends on the board and available tools (e.g., OpenOCD, J-Link). Similar to flashing, the debug session is associated with the selected build directory.

In addition to `west debug`, external tools such as `probe-rs` can also be used.

```bash
probe-rs download --chip <chip_name> build_stm32/zephyr/zephyr.elf
```

> For debugging, the current workflow typically relies on IDE integration (e.g., VSCode with probe-rs DAP support) or GDB server mode, rather than using the standalone `probe-rs debug` command directly.
>
> This reflects a shift toward a more integrated debugging experience, where `probe-rs` acts as the debug backend while the frontend is provided by IDEs or standard debugging tools.

This approach is often more flexible, especially when the board does not officially support a specific runner in Zephyr.

From a workflow perspective, Zephyr debugging is conceptually similar to embedded debugging in QNX or Linux-based systems, where the debugger (e.g., GDB) connects to a target via a transport layer (JTAG/SWD), and the ELF file provides symbol information.

In practice, `west debug` is convenient for standard workflows, while direct tools like `probe-rs` are useful for advanced use cases or when dealing with newer hardware support.

---

## Device Model: The Core of Zephyr

The most important concept in Zephyr is:

> Hardware is described in devicetree, and instantiated at compile time.

### Flow

```text
Devicetree → Driver → DEVICE_DT_DEFINE → Device instance → Application
```

### Example (simplified driver)

```c
#define DT_DRV_COMPAT vendor_foo

static int foo_init(const struct device *dev)
{
    return 0;
}

#define FOO_INIT(inst) \
    DEVICE_DT_INST_DEFINE(inst, foo_init, NULL, NULL, NULL, \
        POST_KERNEL, CONFIG_KERNEL_INIT_PRIORITY_DEVICE, NULL);

DT_INST_FOREACH_STATUS_OKAY(FOO_INIT)
```

---

## Application Access Pattern

Unlike Linux:

```c
open("/dev/spi0", ...)
```

Zephyr uses:

```c
const struct device *dev = DEVICE_DT_GET(...);
spi_transceive(dev, ...);
```

### Key takeaway

> Zephyr does not use file-based abstraction.
>
> It uses subsystem-specific APIs (SPI, I2C, GPIO).

---

## Zephyr vs Linux Driver Model

### Linux

```text
DT → match → probe() → runtime device creation
```

### Zephyr

```text
DT → compile-time instance → init at boot
```

### Key difference

| Feature          | Linux   | Zephyr       |
| ---------------- | ------- | ------------ |
| Device discovery | runtime | compile-time |
| probe()          | yes     | no           |
| hotplug          | yes     | no           |

---

## Userspace vs Kernel Space

By default:

* Zephyr runs in **single address space**
* No syscall boundary
* No process isolation

But:

```conf
CONFIG_USERSPACE=y
```

can enable limited userspace (rarely used on MCU).

---

## Zephyr vs Bare Metal vs Embassy

### Zephyr

```text
App → driver API → driver → HW
```

### Bare-metal / HAL / Embassy

```text
App → HAL → HW
```

### Linux

```text
App → syscall → driver → HW
```

---

### Comparison

| Aspect           | Zephyr     | Embassy |
| ---------------- | ---------- | ------- |
| Driver model     | yes        | no      |
| Devicetree       | yes        | no      |
| Hardware control | abstracted | direct  |
| Bring-up cost    | lower      | higher  |

---

## Observations on STM32H7S78-DK

* Basic peripherals (GPIO/UART/SPI) work well
* Advanced features (external flash, LCD, Ethernet) require additional work
* Zephyr handles most SoC init, but not everything

Compared to bare-metal:

> Zephyr significantly reduces bring-up complexity, but still requires understanding of hardware initialization for advanced use cases.

---

## Key Takeaways

### Zephyr is NOT Linux

* No process model
* No VFS
* No syscall

### But it borrows Linux design patterns

* Devicetree
* Kconfig
* Driver abstraction

> Zephyr borrows Linux’s architecture, not its operating model.

### Compile-time system

* No dynamic probe
* No hotplug
* Deterministic initialization

### Best use case

* MCU platforms
* Structured embedded systems
* Driver + BSP development

## Final Thoughts

From an embedded platform perspective:

* Zephyr provides a **structured alternative to bare-metal development**
* It introduces **discipline (driver model + DTS)** without heavy OS overhead

For engineers with Linux/QNX background:

> Zephyr feels like a “static, RTOS-oriented version of a Linux kernel driver world”.

---

## Next Steps

Planned explorations:

* Custom driver (SPI/I2C)
* External flash execution (XIP)
* Comparing Zephyr vs Rust Embassy in real projects

---
