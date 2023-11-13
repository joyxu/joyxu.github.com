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

由于该技术引入了一个新的zone,`ZONE_EXMEM`的概念，在社区里面引起了比较大的争议，主要认为zone是内核内部的概念，不应该暴露在
核心代码以外，CXL内存用`ZONE_MOVABLE`就可以了，增加zone只会增加复杂性，并不会带来任何新的东西。
但是SMDK的maintainer认为，现有的zone的概念不能正确表达CXL内存设备的特点，`ZONE_NORMAL`不允许插拔，`ZONE_MOVABLE`不允许固定，`ZONE_DEVICE`不允许进行页面分配。


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

# 基于PMEM的扩展

虽然intel已经放弃了PMEM，但是`PMEM.io`仍在发展，很多原先基于PMEM的技术，仍然适用于CXL内存设备。
在CXL官网上也重点介绍了CXL内存技术，而且这个技术栈似乎也在成为主流，该技术把CXL内存也作为两种设备对上层呈现：

* 类似系统内存模式，仅仅创建内存、构造cpuless numa节点的模式，用户态程序可以不用修改
* 继承pmem的设备，对外呈现`/dev/dax`设备节点的模式，用户态基于已有的libmemkind从该设备分配CXL内存

## 整体架构

![PMEM CXL overview](/images/cxl_pmem_overview.png)

## devdax使用模式

![PMEM CXL devdax usage](/images/cxl_pmem_devdax_usage.png)

# 参考

* [Memory-management changes for CXL](https://lwn.net/Articles/931416/)
* [RFC - Coherent Device Memory](https://lwn.net/Articles/720380/)
* [Define coherent device memory node](https://lwn.net/Articles/713035/)
* [EXPLORING THE SOFTWARE ECOSYSTEM FOR COMPUTE EXPRESS LINK (CXL) MEMORY](https://pmem.io/blog/2023/05/exploring-the-software-ecosystem-for-compute-express-link-cxl-memory/)
* [把THP的tail page保存到/dev/dax这种慢速内存设备上](https://lore.kernel.org/linux-mm/20210325230938.30752-1-joao.m.martins@oracle.com/)
* [Linux Kernel中AEP的现状和发展](https://cloud.tencent.com/developer/article/1442273)
* [MEMKIND SUPPORT FOR KMEM DAX OPTION](https://pmem.io/blog/2020/01/memkind-support-for-kmem-dax-option/)
* [Compute Express Link(CXL), the next generation interconnect -- Overview and the status of Linux](https://www.fujitsu.com/jp/documents/products/software/os/linux/catalog/NVMSA_CXL_overview_and_the_status_of_Linux.pdf)
* [CXL device (type 3 memory) hotplug](https://lore.kernel.org/linux-cxl/646e7f96f33e2_33fb3294c1@dwillia2-xfh.jf.intel.com.notmuch/T/#m8f3dd3d916600cce4ea088588add5501870c5241)
* [CXL 1: Management and tiering](https://lwn.net/Articles/894598/)
* [CXL 2: Pooling, sharing, and I/O-memory resources](https://lwn.net/Articles/894626/)
