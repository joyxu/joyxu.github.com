---
title: Linux物理内存管理
author: Joy Xu
date: 2023-01-13 01:14:53
tags: [Linux Kernel, memory]
---

物理内存管理涉及到物理内存的上报发现、分配、反碎片。Linux Kernel中相关的概念有node，pglist_data, zone, mem_block等。

# 物理内存上报和发现

一般内存物理部署结构如下图，内存可能连在北桥上，也可能直接连到socket上，所有内存都能被socket共用。

![physical memory arch](/images/memory_physical_arch.png)

物理内存的描述一般用DeviceTree,或者UEFI的`get_memory_map`以及SRAT表传递给Linux Kernel。
在Linux Kernel中，即使以ACPI启动的方式，按照[UEFI的启动协议](https://kernel.org/doc/html/latest/arm64/booting.html)，
UEFI也会准备好一个DeviceTree Blob，并把这个DTB在内存中的物理地址给到Linux Kernel，
Linux Kernel通过DTB中的`memory`节点得到最终的物理内存信息。

		memory@80000000 {
			device_type = "memory";
			reg = <0x00000000 0x80000000 0 0x80000000>;
			/* DRAM space - 1, size : 2 GB DRAM */
		};

详细的可以参考UEFI代码，或者(QEMU代码)[https://github.com/qemu/qemu/blob/master/hw/arm/virt.c#L2322],或者Linux Kernel的Stub代码。

# FixedMap

但是这时候，Linux Kernel还没有建立物理地址到虚拟地址的映射，于是FixedMap机制被引入进来了。
所谓Fixedmap，指的是虚拟地址中的一段区域，该区域中所有线性地址在编译阶段就已经确定好了，在启动阶段，内核会把这些虚拟地址映射到物理地址上。
具体可以参考[fixmap.h](https://elixir.bootlin.com/linux/v6.1/source/arch/arm64/include/asm/fixmap.h#L103)代码。
建立好fixedmap，并读取解析DTB文件，找到物理内存节点，建立struct memblock的整个过程如下:

![FixedMap DTB Memoryblock](/images/fixedmap_dtb.png)

这个过程结束后，已可以通过`memblock_allock`分配物理内存了，但是还不能通过虚拟地址来访问这块物理内存，接下来就是建立虚拟地址到物理地址的映射了。

# 建立页表

物理内存找到后，就要开始建立页表了，这个过程发生在`paging_init`，'map_kernel'和`map_mem`几个函数中。
其中`map_kenrel`是完成kernel各个段的映射，虚拟地址的信息也可以从System.map或者vmlinux中查到。
而`map_mem`则完成其它内存物理地址到虚拟地址的映射。
这两个函数都是通过`__create_pgd_mapping`创建页表，建立的映射。

# 参考

* [Linux物理内存初始化](https://www.cnblogs.com/LoyenWang/p/11440957.html）
* [Linux Memory Managment Frequently Asked Questions](https://landley.net/writing/memory-faq.txt)
* [Physical Memmory Management](https://slideshare.net/AdrianHuang/presentations)
* [ARM64 Kernel Image Mapping的变化](http://www.wowotech.net/memory_management/436.html)
* [内存开机在干嘛？ -memblock](https://www.jianshu.com/p/20e8fec48419)
* [Linux内存管理(四)：paging_init分析](https://blog.csdn.net/yhb1047818384/article/details/109169979?spm=1001.2014.3001.5501)
