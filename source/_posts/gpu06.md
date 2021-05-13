---
title: Linux GPU系列-06-MESA Gallium Virtio GPU驱动
author: Joy Xu
date: 2021-05-13 11:27:18
tags: [linux, gpu]
---

## Virtio GPU 驱动

从用户态到kernel完整过程如下图

![Vitio GPU驱动](/images/mesa_virtiogpu_driver.jpg)

拆开看的话，分成几部分：
* mesa中的用户态驱动：`src/gallium/drivers/virgl`，实现了pipe driver和winsys，但是没实现Vulkan的支持
* guest kernel驱动： `drivers/gpu/drm/virtio`
* host用户态QEMU设备模拟：`hw/display/virtio-gpu-3d.c`,用户态再调用libvirglrender.so，直接透传Open GL命令
* host libvirglrender: 这是单独一套代码，也是freedesktop维护，地址在 https://cgit.freedesktop.org/virglrenderer, 它会再调用host上mesa
* host kernel GPU驱动：取决于机器的实际代码，一般在 `drivers/gpu/drm/` 目录

## 参考

[Mesa代码解读 - OpenGL Call Trace based on VirtioGPU](https://www.shangmayuan.com/a/1047d00755fb45b389014d33.html)
