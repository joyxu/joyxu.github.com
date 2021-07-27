---
title: AMD GPU 分析
author: Joy Xu
date: 2021-05-28 08:29:05
tags: [linux, gpu]
---

## AMD gpu 介绍

AMD的GPU基本上都是独立显卡，自带显存，所以处理起来会比intel的集显复杂些，
会涉及到显存和内存间的同步等操作。

## AMD 独显工作方式

也和之前介绍的gpu工作方式基本一致，还是准备数据给GPU硬件。

![amd gpu_workflow](/images/gpu_amd_workflow.png)

## AMD GPU内存管理

独显和集显最大的一个区别就是内存的管理，涉及到CPU访问GPU的显存，GPU访问系统内存，以及两个内存间的同步等问题，
还包括基本的从哪里分配，释放和SWAP等等。

![amd gpu_mem](/images/gpu_amd_mem.png)


## 源码

由于AMD有独立显存，所以会使用TTM作为GEM的后端管理显存。

kernel的驱动代码已经支持到了AMD的Navi 1X(RDNA)系列，最新的Navi 2X（RDNA2）还不支持。


## 参考

* [DRM框架分析（三）：显存管理](https://crab2313.github.io/post/drm-vram/)
* [Linux环境下的图形系统和AMD R600显卡编程(5)——AMD显卡显命令处理机制](https://www.cnblogs.com/shoemaker/p/linux_graphics05.html)
* [User Guide for AMDGPU Backend](https://llvm.org/docs/AMDGPUUsage.html)
* [AMD GPU虚拟化的按需调度策略](https://navycloud.github.io/2018/07/27/on-demand-scheduling/)
* [AMD显卡的硬件虚拟化核心功能---GPU抢占](https://navycloud.github.io/2017/08/05/gpu-preemption-in-virtualization/)
* [AMD GPU任务调度（1）—— 用户态分析](https://blog.csdn.net/huang987246510/article/details/106658889)
* [GPU初始化和启动流程（r600）](https://blog.csdn.net/jiangbo1017/article/details/51065118)
* [Linux DRM (GPU drivers)](https://adrian.geek.nz/graphics_docs/DRM.html)
* [AMD graphics](https://linuxreviews.org/AMD_graphics)
* [AMDGPU gentoo wiki](https://wiki.gentoo.org/wiki/AMDGPU)
