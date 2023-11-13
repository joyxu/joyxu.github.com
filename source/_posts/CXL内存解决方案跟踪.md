---
title: CXL内存解决方案跟踪
author: Joy Xu
date: 2023-11-13 03:24:00
tags: [Linux Kernel, CXL, memory]
---

随着新型CXL内存设备的出现，Linux Kernel中出现了很多内存技术相关的变化，比如：

* 基于种内存设备类型的Tiered Memory
* 用户态内存监控管理机制DAMON

更多的可以参考最近几年的LSFMM议题，特别是2023年的，LWN也专门做了一个专题[The 2023 LSFMM+BPF Summit](https://lwn.net/Articles/lsfmmbpf2023/)

# SMDK

三星是内存和nvme的大厂，从设备厂商的角度，自然就推了一套结合自己设备的竞争力方案[SMDK](https://github.com/OpenMPDK/SMDK)。
整个方案简单讲，提供了两种使用CXL内存设备的方式：

* 用户态不感知CXL内存设备，保证兼容原有用户态程序，通过C库和内核来做CXL内存和普通DDR的管理控制
* 用户态感知CXL内存，需要显式传递 `MAP_EXMEM` 给`mmap`

整体架构如下：

![SMDK overview](/images/cxl_smdk_overview.png)

一层细节如下:

![SMDK overview2](/images/cxl_smdk_overview2.png)

# 参考
