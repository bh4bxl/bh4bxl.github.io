---
title: 在 Raspberry PI 4 上用 U-Boot 启动
date: 2023-10-26
tags: [Raspberry PI 4, U-Boot]
---

在 RPI4 上用 U-Boot 启动，可以方便的更改启动 kernel 。

## 下载编译 U-Boot

当前 U-Boot 已经支持 RPI4 ，不需要作任何的修改，就可编译部署到板子上。

### 下载源码

可以从 github.com 上直接下载 U-Boot 的源码：

```bash
$ git clone https://github.com/u-boot/u-boot.git
$ git checkout v2023.10
```

这里使用“2023.10”版本。

### 编译

在 PC 上编译 RPI4 的 U-Boot 需要用到交叉编译工具链。

```bash
$ make CROSS_COMPILE=aarch64-linux-gnu- rpi_4_defconfig
$ make CROSS_COMPILE=aarch64-linux-gnu- -j8
```

编译成功后，在 U-Boot 的根目录下会生成``u-boot.bin``等文件。

## 部署 U-Boot

### 复制文件

把生成的``u-boot.bin``文件复制到 RPI4 系统盘的 boot 分区的根目录下。

### ``config.txt``文件

在``config.txt``文件加入以下的内容：

```
kernel=u-boot.bin
enable_uart=1
arm_64bit=1
```

这时候，上电就可以引导到 U-Boot 的命令行了。但是还不能引导到 Linux kernel ，还需要添加启动脚本。

### 启动脚本

创建一个文件 ``boot.source``：

```
setenv kernel_comp_addr_r 0x0A000000
setenv kernel_comp_size 8726865
fdt addr ${fdt_addr} && fdt get value bootargs /chosen bootargs
fatload usb 0:1 ${kernel_addr_r} kernel8.img
booti ${kernel_addr_r} - ${fdt_addr}
```

其中``kernel_comp_size``是``kernel8.img``文件的大小。

使用``mkimage``生成``boot.scr``：

```bash
$ mkimage -T script -A arm64 -C none -a 0x2400000 -e 0x2400000 -d boot.source boot.scr
```

然后把``boot.scr``文件复制到 RPI4 系统盘的 boot 分区的根目录下。

这样就可以启动 Linux 系统了。
