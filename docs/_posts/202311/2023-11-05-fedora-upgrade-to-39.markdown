---
layout: post
title: 升级Fedora到39
date: 2023-11-05
tag: [fedora]
categories: [Linux distribution]
---

Fedora 39已经发布了有一个多星期了，今天尝试把系统升级到最新的39。

## 升级过程

1. 升级之前先运行以下的命令：

    ```bash
$ sudo dnf upgrade --refresh
    ```

    然后重启电脑。

2. 按照system-upgrade插件包：

    ```bash
$ sudo dnf install dnf-plugin-system-upgrade
    ```

    系统已经安装过了就可以跳过。

3. 下载升级包：

    ```bash
$ sudo dnf system-upgrade download --releasever=39
    ```

4. 出现冲突：

    在这一步遇到了问题，出现了以下的错误提示：

    ```
Error: 
 Problem 1: problem with installed package libswscale-free-6.0-5.fc38.x86_64
  - package ffmpeg-libs-6.0-16.fc39.x86_64 from rpmfusion-free conflicts with libswscale-free provided by libswscale-free-6.0-11.fc39.x86_64 from fedora
  - package ffmpeg-libs-6.0-16.fc39.x86_64 from rpmfusion-free conflicts with libswscale-free provided by libswscale-free-6.0-11.fc39.x86_64 from fedora-modular
  - package ffmpeg-libs-6.0-16.fc39.x86_64 from rpmfusion-free conflicts with libswscale-free provided by libswscale-free-6.0-12.fc39.x86_64 from updates
  - package ffmpeg-libs-6.0-16.fc39.x86_64 from rpmfusion-free conflicts with libswscale-free provided by libswscale-free-6.0-11.fc39.x86_64 from updates-modular
  - package ffmpeg-6.0-16.fc39.x86_64 from rpmfusion-free requires ffmpeg-libs(x86-64) = 6.0-16.fc39, but none of the providers can be installed
  - libswscale-free-6.0-5.fc38.x86_64 from @System  does not belong to a distupgrade repository
  - conflicting requests
 Problem 2: problem with installed package firefox-119.0-2.fc38.x86_64
  - conflicting requests
  - package ffmpeg-libs-6.0-16.fc39.i686 from rpmfusion-free conflicts with libavcodec-free provided by libavcodec-free-6.0-11.fc39.x86_64 from updates-modular
  - package ffmpeg-libs-6.0-16.fc39.x86_64 from rpmfusion-free conflicts with libavcodec-free provided by libavcodec-free-6.0-11.fc39.x86_64 from updates-modular
  - problem with installed package libavcodec-free-6.0-5.fc38.x86_64
  - package ffmpeg-libs-6.0-16.fc39.i686 from rpmfusion-free conflicts with libavcodec-free provided by libavcodec-free-6.0-11.fc39.x86_64 from fedora-modular
  - package ffmpeg-libs-6.0-16.fc39.x86_64 from rpmfusion-free conflicts with libavcodec-free provided by libavcodec-free-6.0-11.fc39.x86_64 from fedora-modular
  - package ffmpeg-libs-6.0-16.fc39.i686 from rpmfusion-free conflicts with libavcodec-free provided by libavcodec-free-6.0-11.fc39.x86_64 from fedora
  - package ffmpeg-libs-6.0-16.fc39.x86_64 from rpmfusion-free conflicts with libavcodec-free provided by libavcodec-free-6.0-11.fc39.x86_64 from fedora
  - package ffmpeg-libs-6.0-16.fc39.i686 from rpmfusion-free conflicts with libavcodec-free provided by libavcodec-free-6.0-12.fc39.x86_64 from updates
  - package ffmpeg-libs-6.0-16.fc39.x86_64 from rpmfusion-free conflicts with libavcodec-free provided by libavcodec-free-6.0-12.fc39.x86_64 from updates
  - libavcodec-free-6.0-5.fc38.x86_64 from @System  does not belong to a distupgrade repository
  - firefox-119.0-2.fc38.x86_64 from @System  does not belong to a distupgrade repository
(try to add '--allowerasing' to command line to replace conflicting packages or '--skip-broken' to skip uninstallable packages)
    ```

    用以下的方法解决：

    ```bash
$ sudo dnf sudo dnf install ffmpeg libavcodec-freeworld --best --allowerasing
    ```

    然后在重新下载（``system-upgrade download``）就可以了。

5. 询问导入新的GPG key的时候，输入``y``就可以了。

6. 重启，完成升级：

    ```bash
$ sudo dnf system-upgrade reboot
    ```

    过程需要重启两次，然后就可以进入Fedora 39系统了。

## 升级以后

### 检查版本

```bash
$ cat /etc/fedora-release
Fedora release 39 (Thirty Nine)
```

### EMACS with doom

Doom需要“sync”一下：

```bash
$ ~/.config/emacs/bin/doom sync
```

暂时还没发现其他问题，等有了问题再来记录吧。

## 参考

- [1][Upgrading Fedora Using DNF System Plugin](https://docs.fedoraproject.org/en-US/quick-docs/upgrading-fedora-offline/)

