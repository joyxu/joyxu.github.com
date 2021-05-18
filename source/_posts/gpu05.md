---
title: Linux GPU系列-05-MESA架构
author: Joy Xu
date: 2021-05-13 03:19:55
tags: [linux, gpu]
---

代码分析本应该自下而上的，但是GPU的主要驱动主要在用户态，所以这次从用户态先开始。
而且Frame怎么画出来的，也是由用户设置state和调用draw call产生的。
下图是从[DirectX Spec](https://microsoft.github.io/DirectX-Specs/d3d/CPUEfficiency.html)官网来的：

![DirectX workgroup](/images/d3d_workgroup.png)

## MESA

[MESA源码](https://gitlab.freedesktop.org/mesa/mesa)里面有2套架构，现在驱动主要基于Gallium架构。

![MESA架构](/images/mesa_arch1.png)

Gallium架构

![MESA Gallium架构](/images/mesa_arch2.png)

MESA其实实现了很多API接口，不只是上图的OpenGL、GLX、还有OpenGL ES、OpenVG、OpenCL、VDPAU和OpenMAX等。

当前MESA驱动实现状态大致如下:

![MESA驱动状态](/images/mesa_arch3.png)

![MESA驱动状态2](/images/mesa_arch4.png)

## Gallium展开

Gallium中主要包含下面几块：

* Auxiliary模块：一些公共函数或者辅助的服务，比如状态缓存和缓存管理等。
* State Tracker: 它负责把上层库的state（blend modes、texture statte等）和draw command（glDrawArrays、glDrawPixels）等转换成MESA内部的pipe对象和操作。
* Pipe Driver: 把Gallium的state、shader和primitive概念转换成硬件能懂的语言
* Winsys: 实例化state tracker和pipe driver，并和具体的操作系统及2D显示驱动绑定

其中只有Pipe Driver和具体的硬件有关系。
对于GPU硬件厂家来说，主要实现Pipe driver和winsys层。

从数据流看，如下：

	state tracker -> Pipe Driver -> Winsyst -> 具体的OS -> OS内核态GPU驱动 -> 具体的GPU硬件

另外Gallium中还有两个比较重要的概念：

* CSO context：GPU中有些状态是不变的常量，Gallium提供了CSO机制帮助创建、销毁、管理这些对象
* Draw: 有些硬件不支持坐标变换，光照、裁剪，Gallium也提供了软件机制帮忙实现

加上这两个概念之后，数据流如下：

	state tracker -> CSO context -> Draw -> Pipe Driver -> Winsys -> 具体的OS -> OS内核态GPU驱动 -> 具体的GPU硬件

state tracker和pipe driver通信是通过pipe context 和pipe screen这两个软件概念
state tracker也可以直接通过p_winsys和Winsys通信,一般GPU厂商也要实现一个自己的winsys，可以参考纯软的 `kms_dri_sw_winsys.c`

## 参考

[OpenGL Concepts](https://www.khronos.org/opengl/wiki/Portal:OpenGL_Concepts)
[Gallium3D Technical Overview](https://www.freedesktop.org/wiki/Software/gallium/)
[Nouveau: Accelerated Open Source driver for nVidia cards](https://nouveau.freedesktop.org/)
[How to start developing on Gallium?](https://mesa-dev.freedesktop.narkive.com/X4wQjx8E/how-to-start-developing-on-gallium)
[Linux graphic stack](https://www.studiopixl.com/2017-05-13/linux-graphic-stack-an-overview.html)
[mesa 框架与目录结构](https://winddoing.github.io/post/39ae47e2.html)
[gallium offcial document](https://gallium.readthedocs.io/en/latest/)
[MESA wiki不用翻墙](http://oer2go.org:81/wikipedia_en_all_novid_2017-08/A/Mesa_(computer_graphics).html)
[The Linux graphics stack, Optimus and the Nouveau driver](http://phd.mupuf.org/files/kr2014.pdf)
[Linux Graphics](https://wdv4758h.github.io/notes/graphics/introduction.html)
[Gallium3D](https://akademy2008.kde.org/conference/slides/zack-akademy2008.pdf)
[TG-Gallium Driver Stack](https://wiki.freedesktop.org/www/Software/gallium/gallium3d-xds2007.pdf)
