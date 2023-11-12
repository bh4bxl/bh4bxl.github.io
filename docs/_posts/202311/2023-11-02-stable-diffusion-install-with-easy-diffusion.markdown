---
layout: post
title: 试用Stable Diffusion绘图
date: 2023-11-02
tags: [ai]
categories: ai
---

最近AI的各种应用非常火爆，看到了不少用AI制作的能以假乱真的各种图片，甚至“蛤蟆丝”都在用AI作图以赢得世界的同情。

网上搜索了一下，找到一个比较火的开源AI绘图是“Stable Diffusion”。

“Stable Diffusion”是2022年发布的深度学习文本到图像生成模型。它主要用于根据文本的描述产生详细图像，尽管它也可以应用于其他任务，如内补绘制、外补绘制，以及在提示词指导下产生图生图的转变。”[1]

开源的，网上教程也比较丰富，于是就试用了以下。

## 安装

SD的是用Python写的，需要用到PyTorch库，以及GPU的支持，依赖的包比较多，安装相对比较麻烦。但是有很多的第三方提供了不少脚本，使得安装更容易一些。

这里选用了[Easy Diffusion](https://easydiffusion.github.io/)提供的脚本。安装脚本可以在[这里](https://github.com/easydiffusion/easydiffusion/releases)下载。

安装的环境是：OS，Fedora 38；HW，Intel 11Gen i7 + Nvidia 2060。

下载最新的Linux包（当前版本是3.0.6），然后解压缩，把``easy-diffusion``复制到用户的``$HOME``目录下，然后进入这个目录运行脚本进行安装：

```bash
$ ./start.sh
```

第一次运行这个脚本会把所有的依赖宝都安装到这个目录（``easy-diffusion``）下，包括Conda、PyTorch等。

等下载和安装结束后，会自动在浏览器中打开网址：“http://localhost:9000/”，这个就是SD的Wed界面。

## 简单使用

启动SD只要在Easy Diffusion的安装目录下（例如：``easy-diffusion``）下运行``./start.sh``就可以了。Web浏览器会自动打开Web UI的。

左边是Prompt和图像的各种参数设置，等以后研究明白了再记录下来吧。

右边是生成的图片。点击“Make Image”按钮即可在右边生成图片。

### 模型安装

安装完成后，Easy Diffusion自带一个Base模型：sd-v1-5。如果要作更多的图，这个模型有点不够，还需要添加更多的模型。

模型可以到以下的网站下载：

- [CIVITAI](https://civitai.com/)
- [哩布哩布AI](https://www.liblib.ai/)

下载的模型有如下的几种：

| 类型         | 后缀                     | 大小 | 说明                     | 存放目录                                   |
|--------------|--------------------------|------|--------------------------|--------------------------------------------|
| CheckPoint   | .ckpt, .safetensors      | 2-7G | Base模型                 | ``easy-diffusion/models/stable-diffusion`` |
| Lora         | .ckpt, .safetensors, .pt | <1G  | 微调模型                 | ``easy-diffusion/models/lora``             |
| VAE          | .ckpt, .pt, 名字带有ae   |      | 美化模型                 | ``easy-diffusion/models/vae``              |
| Embeddings   | .pt                      | <1M  | 嵌入模型，用来调教文本   | ``easy-diffusion/models/embeddings``       |
| Hypernetwork | .pt                      | <1M  | 超网络模型，用于定制画风 | ``easy-diffusion/models/hypernetwork``     |

模型下载好后，放到相应的目录下即可。在“Image Settings”里面可以进行选择。

Stable Diffusion还有很多插件，等以后研究明白了再记录下来吧。

## 参考

- [1][Wikipedia:Stable Diffusion](https://zh.wikipedia.org/zh-cn/Stable_Diffusion)

