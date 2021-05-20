---
title: Linux GPU系列-03-图形软件栈
author: Joy Xu
date: 2021-05-11 09:14:56
tags: [linux, gpu]
---

# Linux类系统上的图形软件栈

图形软件堆栈包括窗口系统和3D显示两部分。
不同的操作系统上图形软件栈也不一样，甚至在linux类系统上，由于GPU硬件厂商的原因，软件栈也不尽相同。
本文主要介绍Linux类操作系统上的图形软件栈。

# 窗口系统

窗口系统是最常见的人机交互界面，比如Windows系统上的GUI系统，打开一个Word文档等都是和窗口系统交互。
在Linux类操作系统上主流的GUI有GNOME，XFCE（它两都基于GTK），KDE（基于QT），国产麒麟系统的UKUI
（基于GNOME），Deepin的DDE（基于QT）。

Linux的窗口系统比较特别，上面这些常见的GUI其实都是基于窗口系统协议上的，当前主要有2类协议：
* X11
* Wayland

这两个协议均由freedesktop维护，Wayland比较新，且在主流的商业发行版中，X11基本被Wayland取代。
Ubuntu从17.10开始支持Wayland，21.04默认使用Wayland；Redhat从8.0版本开始支持Wayland；Debian从
2019年发布的Debian 10开始支持Wayland。

Wayland只是一个协议，协议的具体实现有好几个，比如Weston、Mutter等。

# 3D显示

和Windows上的Direct3D一样，Linux上做3D也有相应的规范，只是Linux上好几类，比如OpenGL、OpenGL ES、
Vulkan等。

OpenGL和Vulkan这两个规范都由Khronos Group组织维护，同时这个组织还维护了SYCL规范（异构计算），
OpenCL规范、EGL规范（不同操作系统上怎么画图）。其中EGL抽象了一组API，支持在不同操作系统上画图，
是OpenGL和Vulkan和操作系统间的桥梁。

说了这么多规范，谁来实现他们呢？各个GPU厂商其实有自己的实现，比如ARM、AMD和NVIDIA等，也有开源项目
MESA等。

MESA同时实现了Vulkan、OpenGL、OpenGL ES、OpenCL、EGL、WGL、VA、VDPAU等规范，真是我们的好朋友：）
具体的文档可以参考 [官方文档](https://docs.mesa3d.org/)
GPU的用户态驱动也是在MESA在实现，MESA会通过DRM动态库，把硬件的具体操作通过IOCTL提交给内核态的GPU驱动。
内核态的GPU驱动，会把用户态传递过来的操作调度给GPU硬件。

# 软件栈全家桶

完整的软件栈如下图，非常感谢wiki画这个图的兄弟，原始的还是SVG格式。

![GPU arch](/images/gpu-arch.png)

用户态主要维护状态机、渲染context、把高级的shader语言编译成GPU识别的ISA、组装GPU的job或者command。
内核态主要处理和内存、提交GPU的job或者command、调度以及中断，还有和Window System交互。

# 安卓

安卓的软件栈又稍稍不一样，安卓去掉了X11和Wayland，实现了自己的drm_hwcomposer和内存管理gralloc。

第一篇就到这里，先把整体的架构把握住。

# 参考
[The Linux Graphics Stack](https://blog.mecheye.net/2012/06/the-linux-graphics-stack/)
[Xplain](https://magcius.github.io/xplain/article/)
[4 years later](https://www.rojtberg.net/271/4-years-later/)
[Martin Peres](https://publications.mupuf.org/)
[Linux graphic stack](https://www.studiopixl.com/2017-05-13/linux-graphic-stack-an-overview.html)
[Daniel Vetter](https://blog.ffwll.ch/archive/)
[Graphics Presentations](https://elinux.org/Graphics_Presentations)
[Open Source Graphics 101](https://elinux.org/images/4/42/Elce-2019-gfx-101-boris.pdf)
[GPU on KVM](https://www.lse.epita.fr/lse-summer-week-2014/slides/lse-summer-week-2014-24-GPU%20on%20KVM.pdf)
