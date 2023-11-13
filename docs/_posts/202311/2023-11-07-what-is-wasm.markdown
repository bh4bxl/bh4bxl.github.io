---
layout: post
title: 什么是WebAssembly
date: 2023-11-07
tags: [web]
categories: web
---
## 定义

WebAssembly或称wasm是一个低级编程语言。WebAssembly是便携式的抽象语法树，被设计来提供比JavaScript更快速的编译及执行。WebAssembly将让开发者能运用自己熟悉的编程语言（最初以C/C++作为实现目标）编译，再藉虚拟机引擎在浏览器内执行[1]。

在 2019 年 12 月 5 日，W3C制定《WebAssembly核心规范》，WebAssembly 正式被认证为 Web 的标准之一[1]。

## WebAssembly是干什么用的

Web上已经有了JavaScript等技术了，为什么还需要WebAssembly呢？

JavaScript虽然在各种的浏览器的支持下，性能虽然得到了巨大的提升，但是还是不能满足更多的性能需求，例如运行需要OpenGL支持的游戏等。

WebAssembly就是针对性能发展出的技术。

WebAssembly是二进制字节码，嵌入在网页里面，通过浏览器的支持，可以以接近原生的速度在本地CPU上执行。同时JavaScript也可以通过API进行调用。

WebAssembly二进制文件可以使用高级语言（例如C++、Rust等）编译得到，使得以前开发的一些应用能够快速的移植到Web上。例如现在一些原来用C++编写的游戏，很多都有可能在网页中运行。

一个基于Unity写的Demo：[Tanks](https://www.wasm.com.cn/demo/Tanks/)

有了WebAssembly加持，Web上支持FFMpeg，OpenCV等技术都是可以预期的。

## 用C++开发一个简单的WebAssembly网页

首先需要有一个WebAssembly的开发编译环境。

### 搭建WebAssembly开发环境

下载Emscripten：

```bash
$ git clone https://github.com/juj/emsdk.git
```

编译Emscripten：

```bash
$ cd emsdk
$ ./emsdk install latest
$ ./emsdk activate latest
```

### 一个简单的C++例子

首先要Enable Emsdk环境：

```bash
$ source ./emsdk_env.sh
```

验证环境：

```bash
$ emcc --version                        
emcc (Emscripten gcc/clang-like replacement + linker emulating GNU ld) 3.1.48 (e967e20b4727956a30592165a3c1cde5c67fa0a8)
...
```

创建一个简单的C++文件：

```c++
#include <iostream>

int main(void) {
    std::cout << "Hello, wasm!" << std::endl;
    return 0;
}
```

编译：

```bash
$ emcc hello.cpp -s WASM=1 -o hello.html
```

生成以下的文件：

```bash
$ ls
hello.cpp  hello.html  hello.js  hello.wasm
```

其中：
- ``hello.wasm``就是生成的WebAssembly二进制字节码
- ``hello.js``包含了用来在原生C/C++函数和JavaScript/wasm之间转换的胶水代码的JavaScript文件
- ``hello.html``用来加载、编译、实例化wasm代码并且将它输出在浏览器显示上的一个HTML文件

### 运行网页

显示WebAssambly的网页，需要有一个支持WebAssembly的浏览器，并启用WebAssembly。

之间在本地通过``file://``方式打开生成的HTML文件会出错，需要通过HTTP服务器运行HTML文件。

如果安装了Python的HttpServer模块，可以直接在本地启动一个Web Server用于测试生成的HTML文件。

```bash
$ python3 -m http.server
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

然后在浏览器中输入``http://localhost:8000/hello.html``，在页面上的Emscripten控制台和浏览器控制台中看到“Hello, wasm!”的输出。

用Emscripten自带的``emrun``也可以检验``hello.html``的运行效果：

```bash
$ emrun hello.html 
Now listening at http://0.0.0.0:6931/
Opening in existing browser session.
```

## 参考

- [1][Wikipedia: WebAssembly](https://zh.wikipedia.org/zh-cn/WebAssembly)
- [2][WebAssembly是什么](https://juejin.cn/post/7002151996698411015)
- [3][Mozilla: WebAssembly](https://developer.mozilla.org/zh-CN/docs/WebAssembly)

