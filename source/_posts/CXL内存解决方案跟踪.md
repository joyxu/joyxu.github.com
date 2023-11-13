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

## 代码细节

因为整个代码树已经没有了具体的commit历史信息了，看代码的时候建议以`ZONE_EXMEM`或者`CONFIG_EXMEM`来看，比较方便。

### 初始化

内核中内存初始化的建议流程如下：

		cxl_pci_probe
		  cxl_dvsec_rr_decode
		    register_cxl_dvsec_ranges
		      update_or_add_cxl_meminfo (内存起始地址，范围，numa节点)
		        request_mem_region
		        add_subzone
		        add_memory_driver_managed
		          mem_hotplug_begin
		          memblock_add_node (添加CXL物理内存)
		          __try_online_node
		             hotadd_init_pgdat (建立CXL内存的struct page到物理内存的映射)

# 参考
