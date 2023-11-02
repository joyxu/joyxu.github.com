---
title: Linux物理内存管理
author: Joy Xu
date: 2023-01-13 01:14:53
tags: [Linux Kernel, memory]
---

物理内存管理涉及到物理内存的上报发现、物理内存到逻辑的映射、分配释放、反碎片、迁移等。
Linux Kernel中相关的概念有`node`，`pglist_data`, `zone`, `memblock`，`page`，`page frame`，`mem_section`，``，，等。

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

![memblock boot from acpi](/images/memory_block_acpi_boot.png)

## fixmap

所谓`fixmap`，指的是虚拟地址中的一段区域，该区域中所有地址在编译阶段就已经确定好了，在Linux Kernel启动阶段，
内核会把这些虚拟地址映射到物理地址上。之所以引入`fixmap`，是因为在内核启动过程中，某些模块需要使用虚拟地址，
但是这时候内核还没有完全启动，并没有完成虚拟地址到物理地址映射这个复杂的过程，
所以一个简化的虚拟内存的分配和管理的机制就被引入了，也就是要说的`fixmap`。

比如说启动的时候，虽然传递了DTB的物理地址，但Linux Kernel还没有建立物理地址到虚拟地址的映射，还不能读取，
这时候就可以通过`fixmap`机制，读取解析DTB文件，找到物理内存节点，建立`struct memblock`，
整个流程可以参考`DeviceTree`章节中的流程图。还比如说早期的`early_ioremap`来访问外设的寄存器等。
`fixmap`中各个地址区间的定义可以参考代码[fixmap.h](https://elixir.bootlin.com/linux/v6.1/source/arch/arm64/include/asm/fixmap.h#L103)。

上面这个过程结束后，所有的物理内存都可以通过`memblock`、`memblock_region`等结构体呈现了，示意图如下：

![memblock and region](/images/memory_block.png)

配套的日志和debug接口信息如下:

![memblock and region2](/images/memory_block_dmesg_debug.png)

现在已经可以通过`memblock_allock`分配物理内存了，但是还不能通过虚拟地址来访问这块物理内存，而且cpu、numa和zone
的信息已经可以获取到，接下来就是建立虚拟地址到物理地址的互相映射了，并把物理页面挂到不同cpu节点的zone里面了。

# 物理内存到内核逻辑概念的映射

要想使用物理内存，自然就会想到把这段物理内存地址映射到虚拟地址，于是就有了页表的建立。
另外呢，物理内存一般被划分成`page frame`来管理，对应到内核中的概念`struct page`。
一般一个虚拟地址其实是落到单个`struct page`中的offset，而物理地址同样也会落到单个`page frame`中，
所以虚拟地址到物理地址的转换，其实也可以复用到`page`到`page frame`的映射。

![page frame and page](/images/memory_pfn_page.png)

## 虚拟地址和页表

在ARM64平台上，以4KB页表页表寻址过程如下图:

![page table walking](/images/memory_pagetable_walking.png)

其中table\block\page描述符和虚拟地址格式如下图：

![page table walking](/images/memory_page_descriptions.png)

以一个虚拟地址为例话，地址翻译的过程如下：

![page table walking](/images/memory_pagetable_walking2.png)

## 建立页表

建立页表的输入是`memblock`，也就是物理内存的起始地址和结束地址，输出是空的页表。
建立页表这个过程发生在`paging_init`，`map_kernel`和`map_mem`几个函数中。
其中`map_kenrel`是完成kernel各个段的映射，虚拟地址的信息也可以从System.map或者vmlinux中查到。
而`map_mem`则完成前面发现的`memregion`的物理地址到虚拟地址的映射。这两个函数都是通过`__create_pgd_mapping`创建页表，建立的映射。
这时候最底层的页表并没有pte，pte的创建在发生缺页的时候建立。

具体流程如下：

![page table directory create](/images/memory_pagetable_create.png)

## page frame到page的映射

页表框架搭起来之后，就到了把物理内存转换到内核物理内存逻辑概念的阶段，目前内核管理物理内存有四种模型，但主要使用sparse模型。

![physical memory models](/images/memory_physical_models.png)

以ARM64为例，具体函数在`bootmem_init`，其中：
* `arm64_numa_init` 负责建立numa的cpu节点
* `arm64_memory_prenset` 负责把大的`memblock`、`memblock_region`拆成小的`mem_section`来管理，并和cpu节点关联起来
* `sparse_init` 把物理page frame和`mem_section`、`struct page`关联起来，这样通过物理page frame number就可以找到具体的struct page
* `zone_sizes_init`初始化各个cpu节点的zone信息。

这时候，`memblock`、`memblock_region`、`mem_section`和`struct page`关系如下：

![physical memory model](/images/memory_physical_models3.png)

在这种模型下，物理page frame number转换到struct page的过程如下:

![physical frame to page](/images/memory_pfn2page.png)

一个PFN到Page的运行实例如下图:

![physical frame to page2](/images/memory_pfn2page2.png)

## 物理地址到page frame的转换实例

物理地址到PFN的转换按照前面的介绍，就是物理地址左移`PAGE_SHIFT` page的大小就可以了。以4K为例：

![physical to pfn](/images/memory_phy2pfn.png)

## virtual address到Physical address的映射

一个virtual address到Physical address的运行实例如下图:

![virtual address to physical address](/images/memory_virt2phy.png)

### 为什么内核中有的物理地址对应到了2个虚拟地址

内核中虽然只有一份页表，但是有几种映射关系，这也是kernel logic address(kmalloc)和kernel virtual address(vmalloc)，以及kmap之间的关系。

在[LDD3](https://makelinux.net/ldd3/chp-15-sect-1.shtml)中也有描述：

![address types used in linux](/images/memory_ldr3_1501.gif)

以jiffies为例，在crash工具中，`vtop`通常返回`kernel logic address`，但是可以通过`kmem`看物理地址对应的虚拟地址。

![address types used in linux2](/images/memory_phy2virt2.png)

## 把page加到zone

接下来就是把`struct page`放到各个cpu节点zone下面的`free_area`中，就到了`zone page frame allocator`阶段。
函数调用栈和实例如下：

![physical memory model](/images/memory_physical_models4.png)

到此，物理内存已经可以基于CPU的拓扑结构，在zone的基础上，基于page来分配了，结构如下:

![physical memory model](/images/memory_physical_models2.png)

运行实例图如下:

![physical memory model](/images/memory_physical_models5.png)

后面就是内存伙伴`buddy`系统的事情了。

![zone frame allocator](/images/memory_zone_frame_allocator.png)

# 参考

* [linux内存管理](https://blog.csdn.net/wwwlyj123321/article/details/128241134)
* [内存管理源码分析-内核页表的创建以及索引方式(基于ARM64以及4级页表)](https://blog.csdn.net/u011649400/article/details/105984564)
* [Linux内存管理(三)：“看见”物理内存](https://blog.csdn.net/yhb1047818384/article/details/108328097?spm=1001.2014.3001.5501)
* [Linux内存管理(四)：paging_init分析](https://blog.csdn.net/yhb1047818384/article/details/109169979?spm=1001.2014.3001.5501)
* [Linux物理内存初始化](https://www.cnblogs.com/LoyenWang/p/11440957.html)
* [Linux Memory Managment Frequently Asked Questions](https://landley.net/writing/memory-faq.txt)
* [Physical Memmory Management](https://slideshare.net/AdrianHuang/presentations)
* [setup_arch：bootmem_init : sparse_init](https://blog.csdn.net/jasonactions/article/details/122930298)
* [ARM64 Kernel Image Mapping的变化](http://www.wowotech.net/memory_management/436.html)
* [内存开机在干嘛？ -memblock](https://www.jianshu.com/p/20e8fec48419)
* [Linux内存管理(四)：paging_init分析](https://blog.csdn.net/yhb1047818384/article/details/109169979?spm=1001.2014.3001.5501)
* [Fix-Mapped Addresses](http://www.wowotech.net/memory_management/fixmap.html)
* [linux-kernel-labs](https://linux-kernel-labs.github.io/refs/heads/master/lectures/address-space.html)
* [A little bit about a linux kernel](https://github.com/0xAX/linux-insides)
* [d42_5_overview_of_the_vmsav8-64_address_translation](https://armv8-ref.codingbelief.com/en/chapter_d4/d42_5_overview_of_the_vmsav8-64_address_translation.html)
* [Armv8-A Address Translation](https://documentation-service.arm.com/static/5efa1d23dbdee951c1ccdec5)
* [slideshare 翻墙下载](https://ssslideshare.com/)
* [Linuxでのpage構造体群の配置](https://qiita.com/akachochin/items/121d2bf3aa1cfc9bb95a)
* [Linux Kernel Memory Hacking](https://oliveryang.net/2017/03/linux-kernel-memory-hacking/)
* [内存是怎么映射到物理地址空间的？内存是连续分布的吗？](https://zhuanlan.zhihu.com/p/66288943)
* [Linux crash工具结合/dev/mem任意修改内存](https://blog.csdn.net/dog250/article/details/102704164)
* [ARMv8 MMU及Linux页表映射](https://www.cnblogs.com/LoyenWang/p/11406693.html)
* [用crash tool观察ARM64 Linux地址转换](https://www.cnblogs.com/bigfish0506/p/16273091.html)
* [Memory Management in Linux](https://makelinux.net/ldd3/chp-15-sect-1.shtml)
* [Minimizing struct page overhead](https://blogs.oracle.com/linux/post/minimizing-struct-page-overhead)
