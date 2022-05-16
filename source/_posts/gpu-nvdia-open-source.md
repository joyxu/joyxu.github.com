---
title: gpu-nvdia-open-source
author: Joy Xu
date: 2022-05-16 02:26:20
tags:[linux, gpu, nvdia]
---

## Nvdia GPU开源驱动

nvdia最近开源了 [GPU内核代码](https://github.com/NVIDIA/open-gpu-kernel-modules) , 但遗憾的是用户态代码还未开源。
不管怎么样，对我来说还是好消息。
nvidia这次开源的代码量还是很大，放个细节，大家自己体会下

[!整体代码量](/images/nvdia_code_sum.png)

[!kernel-open下的代码量](/images/nvdia_code_sum2.png)

一般GPU内核驱动主要做几个事情：

* 设备初始化，对接PCIe框架，初始化相关的Bar，提供mmio 寄存器访问;支持PCIe SRIOV/PRI/ATS等特性，划分VF资源，支持虚拟化
* 中断初始化
* 内存管理
* drm框架对接，支持上层mesa用户态驱动调用，包括上下文管理，bo管理，GPU job管理，功耗管理和显示管理

本文从上面几个角度，重点分析Nvdia开源驱动代码。

## 编译

Nvdia驱动的编译方式和自带的README文件都不太友好，特别是交叉编译方式下，并不能完整编译通过。
编译之后，主要有5个KO，都从子目录kernel-open生成：

* nvdia-peermem.ko: peer to peer memory
* nvdia-uvm.ko: unified virtual memory
* nvdia-modeset.ko: 设置显示器的modeset
* nvdia-drm.ko: 对接linux kernel drm框架
* nvdia.ko: 驱动主入口


## nvdia.ko

### 设备初始化

入口函数在nv-frontend.c的nvidia_frontend_init_module中，nv-frontend作为逻辑设备管理的总入口，提供设备的add和remove机制，
并为所有设备文件提供统一的入口操作函数。

每一个逻辑设备又对外表现为两个linux kernel设备：一个字符设备和一个pci设备。

* 字符设备提供nv_fops，用来管理用户态的ioctl和mmap：

[!字符设备ioctl](/images/nvdia_chardev_ioctl.png)

[!字符设备mmap](/images/nvdia_chardev_mmap.png)

* pci设备用来管理功耗nv_pm_ops和出错处理nv_pci_error_handlers

[!pci ops](/images/nvdia_pcidev_ops.png)

### 中断

nvidia支持几种中断方式，本文只以msi/msix为例，入口在nv-msi.c和nv.c中的nvidia_isr_misx函数。
所处理的中断主要有3中类型：rm，vum和rm_fault。

### 内存管理

nvdia内存分按照以下三种形式划分：

[!memory type](/images/nvdia_memory_type.png)


nvdia的内存管理很复杂，把nvlink，nv switch, peertopeer、dmabuf等概念都给串了起来，后面单独用一篇文章分析。

### MTRR & PAT

这两个是intel上内存是否可以cache的特性，而非arm上的特性，暂时不关注，忽略掉nv-pat.c

## nvdia-drm.ko

和amd、intel的驱动不一样，nvidia这块主要是和显示相关的处理，包括crtc、connector、encoder和mode的处理。

## 虚拟化

nvidia的GPU虚拟化支持以下几种方式，当前主要以SRIOV为主：

[!virtualization](/images/nvdia_vgpu_type.png)

由于nv-pci-table.c匹配了nvidia下所有gpu设备，因此vgpu 设备也会被匹配执行`nv_pci_probe`。
但更细节的内容目前的代码还没有提供，比如`nvidia_vgpu_vfio_probe`等函数。
