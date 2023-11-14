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

## system ram使用缺陷

cpuless NUMA节点的形式出现，Linux Kernel就可以分配PMEM上的memory，这样和使用一般DRAM没什么区别，但只是解决了第一步的问题，怎么把PMEM“用好”的问题还没有解决。
比如，当内核分配内存时，如果从PMEM上分配了memory，并且这块内存上的数据是被经常访问的，那么由于物理特性上的差异，一般应用都会体会到性能的下降。
那么怎么更明智的使用PMEM就是一个待解决的问题。

吴峰光的一组patch[PMEM NUMA node and hotness accounting/migration](https://lore.kernel.org/linux-mm/20190202065741.GA1011@xz-x1/T/)来尝试解决这个问题。

这组patch主要提供了下面几个功能：

1 隔离DRAM和PMEM。为PMEM单独构造了一个zonelist，这样一般的内存分配是不会分配到PMEM上的。
2 跟踪内存的冷热。利用内核中已经有的idle page tracking功能（目前主线内核只支持系统全局的tracking），在per process的粒度上跟踪内存的冷热。
3 利用现有的page reclaim，在reclaim时将冷内存迁移到PMEM上（只能迁移匿名页）。
4 利用一个userspace的daemon和idle page tracking，来将热内存（在PMEM上的）迁移到DRAM中。

这组patch发到LKML以后，引来了很激烈的讨论，主要集中在两个方面：

1 为什么要单独构造一个zonelist把PMEM和DRAM分开？

目前的NUMA API只能控制从哪个node分配，但是不能控制比例，比如mbind()，只能告诉进程这段VMA可以用哪些node，但是不能控制具体多少memory从哪个node来。
要想做到更细粒度的控制，需要改造目前的NUMA API。而且目前memory hierarchy越来越复杂，比如device memory，这都是目前的NUMA API所不能很好解决的。

2 能不能把冷热内存迁移通用化？

冷热内存迁移这个方向是没有问题的，问题在于目前patch中的处理太过于PMEM specific了。
内核中的NUMA balancing是把“热”内存迁移到最近的NUMA node来提高性能。但是却没有对“冷”内存的处理。所以能不能实现一种更通用的NUMA rebalancing?
比如，在reclaim时候，不是直接reclaim内存，而是把内存迁移到一个远端的，或者空闲的，或者低速的NUMA node，类似于NUMA balancing所做的，只不过是往相反的方向。
patch[Another Approach to Use PMEM as NUMA Node](https://lore.kernel.org/linux-mm/1554955019-29472-1-git-send-email-yang.shi@linux.alibaba.com/)，就体现了这种思路。
利用Kernel中已经很成熟的memory reclaim路径把“冷”内存迁移到PMEM node中，NUMA Balancing访问到这个page的时候可以选择是否把这个页迁移回DRAM，相当于是一种比较粗粒度的“热”内存识别。

社区中还有一种更加激进的想法就是不区分PMEM和DRAM，在memory reclaim时候只管把“冷”内存迁移到最近的remote node，如果target node也有内存压力，那就在target node上做同样的迁移。
但是这种方法有可能引入一个内存迁移“环”，导致内存在NUMA node中间不停地迁移，有可能引入unbounded time问题。而且一旦node增多，可能会迅速恶化问题。
在内存回收方面还有一个更可能立竿见影的方案就是把PMEM用作swap设备或者swap文件。
目前swap的最大问题就是传统磁盘的延迟问题，很容易造成系统无响应，这也是为什么有zswap这样的技术出现。
PMEM的低延迟特性完全可以消除swap的延迟问题。

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
