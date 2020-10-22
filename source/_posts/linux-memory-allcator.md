---
title: linux memory management
author: Joy Xu
date: 2020-08-21 19:18
tags: [Linux Kernel]
---

先立个Flag，给自己一个Todo :)

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
* high memory: 高地址的128MB,不总是映射到kernel address space，为了在32位系统中
能访问更多的内存，比如64GB(36位)，kernel引入了PAE(page address extension)的概念，
由于36超过32，所以有部分内存不是总被映射，这部分就叫做high memory。但在64位系统中，
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

![x86 zone info](/images/zone-types.jpg]

在ARM64中，没有zone_highmem区域，具体代码参考[zone_type](https://elixir.bootlin.com/linux/latest/source/include/linux/mmzone.h#L345)。

![zone info](/images/zone-info.PNG)

参考：
[Linux内存管理zone_sizes_init](https://www.cnblogs.com/LoyenWang/p/11568481.html)
[linux内核内存管理](https://blog.csdn.net/farmwang/article/details/66976818)
