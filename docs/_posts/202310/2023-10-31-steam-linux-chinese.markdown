---
layout: post
title: 在Linux上运行的Stream正确显示中文
date: 2023-10-31
tag: [steam, chinese]
categories: steam
---

在Fedora Linux上安装了Steam之后，运行Steam，然后设置成中文。重新启动Steam之后，发现菜单和里面的内容，应该显示中文的地方都显示的是一个个方块。

## 解决方法

创建一个文件``steam-fontconfig.conf``，内容如下：

```xml
<?xml version="1.0"?>
<!DOCTYPE fontconfig SYSTEM "fonts.dtd">
<fontconfig>
  <include ignore_missing="no">fonts.conf</include>

  <match target="pattern">
    <test qual="any" name="family">
      <string>Arial</string>
    </test>
    <edit name="family" mode="assign" binding="same">
      <string>WenQuanYi Zen Hei</string>
    </edit>
  </match>
</fontconfig>
```

这里用的是文泉驿正黑字体，也可以使用其他中文字体。

然后用以下办法启动Stream：

```bash
$ FONTCONFIG_FILE=/home/bh4bxl/steam-fontconfig.conf /usr/bin/steam
```

修改Application里面的Steam启动方式：复制``steam-fontconfig.conf``到``/usr/local/etc/``目录下；然后打开``/usr/share/applications/steam.desktop``文件，修改``[Desktop Entry]``下的``Exec``：

```
Exec=env FONTCONFIG_FILE=/usr/local/etc/steam-fontconfig.conf /usr/bin/steam %U
```

这样从菜单里启动的Steam也可以显示中文了。

## 参考

- [1] [Linux下Steam中支持中文的办法] (https://www.cnblogs.com/eaglexmw/p/5560345.html)

