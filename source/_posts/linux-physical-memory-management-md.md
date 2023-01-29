---
title: Linux物理内存管理
author: Joy Xu
date: 2023-01-13 01:14:53
tags: [Linux Kernel, memory]
---

物理内存管理涉及到物理内存的上报发现、分配、反碎片。Linux Kernel中相关的概念有`node`，`pglist_data`, `zone`, `mem_block`等。

# 物理内存上报和发现

一般内存物理部署结构如下图，内存可能连在北桥上，也可能直接连到socket上，所有内存都能被socket共用。

![physical memory arch](/images/memory_physical_arch.png)

物理内存的描述一般用DeviceTree,或者UEFI的`get_memory_map`以及SRAT表传递给Linux Kernel。

## DeviceTree

如果是DeviceTree启动的话，一般通过`memory`节点描述物理内存信息，例子如下:

		memory@80000000 {
			device_type = "memory";
			reg = <0x00000000 0x80000000 0 0x80000000>;
			/* DRAM space - 1, size : 2 GB DRAM */
		};


Linux Kernel启动过程中，DTB的物理地址作为参数传递给内核，我们先跳过`fixmap`的概念，了解下DTB发现物理内存大致流程：

![FixedMap DTB Memoryblock](/images/fixedmap_dtb.png)

## UEFI/ACPI

如果通过UEFI启动的话，Linux Kernel会通过`drivers/firmware/efi/libstub`中的`efi_boot_kernel`函数调用UEFI的`get_memory_map`
找到物理内存相关信息，并按照[UEFI的启动协议](https://kernel.org/doc/html/latest/arm64/booting.html)，准备或者更新DeviceTree，
Linux Kernel通过DTB中的`uefi-mmap-start`节点得到最终的物理内存信息，注意这个地址里面是一个链表，
存放着一组`efi_memory_desc_t`结构体，这个结构体中描述了每片物理内存的物理起始地址和大小，
更详细的可以参考UEFI代码，或者(QEMU代码)[https://github.com/qemu/qemu/blob/master/hw/arm/virt.c#L2322], 或者Linux Kernel的Stub代码。

UEFI启动流程，内存节点上报流程关键调用栈如下:

	efi_boot_kernel
	  efi_bs_call get_memory_map
	    update_fdt
	      start_kernel
	        setup_arch
	          efi_init
	            reserve_regions
	              memblock_add

示例图如下：

![memoryblock boot from acpi](/images/memory_block_acpi_boot.png)

## fixmap

所谓`fixmap`，指的是虚拟地址中的一段区域，该区域中所有地址在编译阶段就已经确定好了，在Linux Kernel启动阶段，
内核会把这些虚拟地址映射到物理地址上。之所以引入`fixmap`，是因为在内核启动过程中，某些模块需要使用虚拟地址，
但是这时候内核还没有完全启动，并没有完成虚拟地址到物理地址映射这个复杂的过程，
所以一个简化的虚拟内存的分配和管理的机制就被引入了，也就是要说的`fixmap`。

比如说启动的时候，虽然传递了DTB的物理地址，但Linux Kernel还没有建立物理地址到虚拟地址的映射，还不能读取，
这时候就可以通过`fixmap`机制，读取解析DTB文件，找到物理内存节点，建立`struct memblock`，
整个流程可以参考`DeviceTree`章节中的流程图，`fixmap`中各个地址区间的定义可以参考代码[fixmap.h](https://elixir.bootlin.com/linux/v6.1/source/arch/arm64/include/asm/fixmap.h#L103)。

上面这个过程结束后，所有的物理内存都可以通过`memblock`、`memblock_region`等结构体呈现了，示意图如下：

![memoryblock and region](/images/memory_block.png)

配套的日志和debug接口信息如下:

![memoryblock and region2](/images/memory_block_dmesg_debug.png)

现在已经可以通过`memblock_allock`分配物理内存了，但是还不能通过虚拟地址来访问这块物理内存，而且cpu、numa和zone
的信息已经建立，接下来就是建立虚拟地址到物理地址的映射了，并把物理页面挂到不同cpu节点的zone里面了。

# 建立页表

建立页表这个过程发生在`paging_init`，`map_kernel`和`map_mem`几个函数中。
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
* [Fix-Mapped Addresses](http://www.wowotech.net/memory_management/fixmap.html)
* [linux-kernel-labs](https://linux-kernel-labs.github.io/refs/heads/master/lectures/address-space.html)
* [A little bit about a linux kernel](https://github.com/0xAX/linux-insides)
