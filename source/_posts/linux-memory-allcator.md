---
title: linux memory management
author: Joy Xu
date: 2020-08-21 19:18
tags: [Linux Kernel, memory]
---

先立个Flag，给自己一个Todo :)

务必先读 gorman的书 [Understanding The Linux Virtual Memory Manager](https://www.kernel.org/doc/gorman/pdf/understand.pdf)
务必先读 gorman的书 [Understanding The Linux Virtual Memory Manager](https://www.kernel.org/doc/gorman/pdf/understand.pdf)
务必先读 gorman的书 [Understanding The Linux Virtual Memory Manager](https://www.kernel.org/doc/gorman/pdf/understand.pdf)

https://landley.net/writing/memory-faq.txt
https://slideshare.net/AdrianHuang/presentations
https://hammertux.github.io/slab-allcator
https://blog.csdn.net/mrwangwang/article/details/38709813
https://inst.eecs.berkeley.edu/~cs162/sp20/static/lectures/13.pdf
https://linuxplumbersconf.org/event/2/contributions/65/attachments/15/171/slides-expanded.pdf
https://manybutfinite.com/post/how-the-kernel-manages-your-memory/
https://www.cse.iitb.ac.in/~mythili/teaching/cs347_autumn2016/notes/07-memory.pdf

# 名词解释

## high memory & low memory

high memory和low memory可以说是针对物理内存的概念，在以前的32位处理器中，kernel
把virtual address space划分成2部分，3G用户空间和1G kernel空间，其中1G的kernel空
间又分成两部分：
* low memory: 低地址的896MB,总是映射到kernel address space。
* high memory: 对于32位系统，1G以外的地址不是全部被映射的，这部分就叫high memory。

为了kernel能访问更多的内存，比如64GB(36位)，kernel引入了PAE(page address extension)的概念，
并预留了kernel address space高地址的104MB用做映射1G以外的内存。但在64位系统中，
high memory也是可以被映射的，但是由于kernel对物理地址连续内存的偏好，以及使用
限制（vmalloc指定GFP_HIGHMEM分配），high memory还是用的不多。 在用户态中，用户地址
是需要显式映射的，通常high memory用于用户态。

参考：
[How Linux kernel decide to which memory zone to use](https://stackoverflow.com/questions/18061218/how-linux-kernel-decide-to-which-memory-zone-to-use)
[Kernel addresses – concept of low and high memory](https://www.oreilly.com/library/view/linux-device-drivers/9781785280009/3ef362cb-6fc3-4089-b7ea-8df1ce77ca5a.xhtml)
[kernel memory](http://iakovlev.org/index.html?m=1&p=1034)

![Kernel addresses – concept of low and high memory](/images/kernel-high-low-memory.PNG)

## zone

kernel使用Zone来管理physical address space range.一般有zone_dma, zone_normal和
zone_highmem，X86上zone的布局可以参考下图：

![x86 zone info](/images/zone-types.jpg)

在ARM64中，没有zone_highmem区域，具体代码参考[zone_type](https://elixir.bootlin.com/linux/latest/source/include/linux/mmzone.h#L345)。

![zone info](/images/zone-info.PNG)

参考：
[Linux内存管理zone_sizes_init](https://www.cnblogs.com/LoyenWang/p/11568481.html)
[linux内核内存管理](https://blog.csdn.net/farmwang/article/details/66976818)

## linux virtual address

linux中virtual address可以有三类：
* Kernel Logical Address: 通过kmalloc分配，和物理地址有一个fixed offset和fixed mapping, 不能被swapped out，和上文的low memory映射。
* Kernel Virtual Address: 也叫vmalloc区域，主要用于映射非连续的物理内存，通过vmalloc分配；以及mmio访问外设，通过ioremap和kmap分配。
* User Virtual Address：用户态虚拟地址，在page_offset之下，每个进程都有自己的映射，一般通过mmu映射，只有使用到的ram才会比映射，一般非连续，可以不swapped out，也可以被move。

![kernel logic address](/images/kernel-logic-address.PNG)

参考：
[Virtual Memory and Linux](https://elinux.org/images/b/b0/Introduction_to_Memory_Management_in_Linux.pdf)

* physmap: physical direct mapping, 用于申请物理地址连续内存,对应上文的kernel logic address。
* vmalloc: dynamic meory region，用于申请虚拟地址连续内存，对应上文的kernel virtual address。
* vmemmap: virtual memory map，专门开辟的一个映射，包含physical page frames的metadata,优化pfn_to_page和page_to_pfn的效率

具体代码[VMEMMAP_START](https://elixir.bootlin.com/linux/latest/source/arch/arm64/include/asm/memory.h#L53)
arm64上映射布局可以参考下图,具体文档[arm64 memory.rst](https://elixir.bootlin.com/linux/latest/source/Documentation/arm64/memory.rst)

![arm64 kernel memory mapping](/images/arm64-kernel-memory-map.png)

由于内核已经把virtual memory layout功能给去掉，从4.15内核里面把[这个代码](https://elixir.bootlin.com/linux/v4.15.18/source/arch/arm64/mm/init.c#L603)加回来之后，
在arm64 virt qemu的机器上，内核的打印如下：

![arm64 kernel memory mapping2](/images/arm64_qemu_virt_memory.png)

## 综述

![mem alloc hierarchy](/images/mem_alloc1.png)

![mem alloc](/images/mem_alloc2.png)

## 内存碎片演进简史

[Linux Kernel vs. Memory Fragmentation](https://en.pingcap.com/blog/linux-kernel-vs-memory-fragmentation-1)

参考：

* [Memory Subsystem and Data Types in the Linux Kernel](https://hps.vi4io.org/_media/teaching/wintersemester_2014_2015/kp-1415-memory-management.pdf)

## linux mm scope

![mm api scope](/images/mm-scope.PNG)

参考：
* [Arm64 Linux Kernel vs. Memory Fragmentation[ARM64 Kernel Image Mapping的变化](http://www.wowotech.net/memory_management/436.html)
* [KASLR-MT: Kernel Address Space Layout Randomization for Multi-Tenant cloud systems](https://github.com/joyxu/archive/blob/master/document/linux/memory/kaslr-mt.pdf)
* [AArch64 Kernel Page Tables](https://wenboshen.org/posts/2018-09-09-page-table.html)
* [Tour de Linux memory management](https://github.com/joyxu/archive/blob/master/document/linux/memory/07_memory_management.pdf)
* [W4118 OPERATING SYSTEMS](http://www.cs.columbia.edu/~junfeng/13fa-w4118/syllabus.html)
* [Minimizing struct page overhead](https://blogs.oracle.com/linux/post/minimizing-struct-page-overhead)
* [struct page, the Linux physical page frame data structure](https://blogs.oracle.com/linux/post/struct-page-the-linux-physical-page-frame-data-structure)
