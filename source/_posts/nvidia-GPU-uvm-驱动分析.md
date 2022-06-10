---
title: nvidia-GPU-uvm-驱动分析
author: Joy Xu
date: 2022-05-24 07:49:03
tags: [linux]
---

## nvidia gpu内存管理架构演进

先看个大图，回忆下nvida GPU架构演进过程中，内存管理相关的演进

![nvidia memory involution](/images/nvidia-memory-roadmap.png)

借用下星辰变境界演进的说法。

## 星云期

最早的时候很简单，只是为了让用户态可以访问GPU的显存，于是gpu驱动就在linux kernel中创建了一个设备，
提供ioctl alloc和mmap操作，用户调用cudamalloc的时候，通过传入该设备的fd，触发设备的alloc和mmap操作，
通过`remap_pfn_range`函数把GPU可访问的物理地址转换成用户态可以访问的va。
通常的写法如下：

![alloc and mmap](/images/alloc-mmap.png)

![alloc and mmap2](/images/mmap.jpg)

这种做法在当前的MESA/DRM中很常见，这里放下AMD GPU的代码，实现很类似：

* [mesa amdgpu alloc and mmap](https://github.com/mesa3d/mesa/blob/main/src/gallium/winsys/amdgpu/drm/amdgpu_cs.c#L731)
* [drm alloc](https://github.com/freedesktop/mesa-drm/blob/master/amdgpu/amdgpu_bo.c#L78)
* [kernel ioctl](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/amd/amdgpu/amdgpu_drv.c#L2674)

放在cuda上，用户态的写法如下：

![nvidia memory cudamalloc](/images/nvidia-memory-cudamalloc.png)

内核态中的代码也很清晰，基本都是在uvm.c中，具体流程如下：

	uvm_mmap_entry
	  uvm_mmap
	    uvm_vma_wrapper_alloc
	      uvm_va_range_create_mmap

## 参考

* [DRM 驱动 mmap 详解：（一）预备知识](https://blog.csdn.net/hexiaolong2009/article/details/107592704)
* [NVIDIA PROFILING TOOLS](https://www.olcf.ornl.gov/wp-content/uploads/2019/08/NVIDIA-Profilers.pdf)
