---
title: Linux GPU系列-01-GPU物理模型
author: Joy Xu
date: 2021-05-09 08:18:04
tags: [linux, gpu]
---

接着上篇，这篇主要讲讲GPU的构成。

# GPU物理模型

GPU通常是一个独立的PCIe卡，当然也可能集成到SoC中，但是仍然呈现为一个PCIe设备。

GPU的物理形态一般如下图：

![GPU卡](/images/gpu_card.png)

具体到GPU的硬件单元的话，以AMD RDNA的卡为例，内部框图如下：

![AMD RDNA GPU](/images/amd_gpu.png)

如果我们把它抽象下，一般可以看成下面的模型：

![GPU模型](/images/gpu_management_model.png)

先说下几个基本概念

## GPU context
GPU context代表GPU当前状态，每个context有自己的page table，多个context可以同时共存

## GPU Channel
每个GPU context都有一个或者多个GPU Channel，CPU发给GPU的命令通过GPU Channel传递，
一般GPU Channel是一个软件概念，通常是一个ring buffer

## page table
和CPU的page table功能一样，用于VA到PA的映射，访问GPU的地址空间

CPU和GPU通信主要有几下几种方式：
* 通过PCIe BAR空间映射出来的寄存器
* 通过PCIe BAR空间把GPU的内存映射到CPU的地址空间中
* 通过GPU的页表把CPU的系统内存映射到GPU的地址空间中
* 通过MSI中断

# 参考

[GPU Architecture Overview](https://insujang.github.io/2017-04-27/gpu-architecture-overview/)
["RDNA 1.0" Instruction Set Architecture](https://developer.amd.com/wp-content/resources/RDNA_Shader_ISA.pdf)
[A deeper look into GPUs and the Linux Graphics Stack](https://phd.mupuf.org/files/toulibre2012_deeper_look.pdf)
