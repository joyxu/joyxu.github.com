---
title: linux kernel地址空间
author: Joy Xu
date: 2022-05-23 02:21:19
tags: [linux]
---

## 开篇立意

异构编程首先要解决的问题是地址空间的问题，CPU的地址，设备的地址是如何映射到各自不同的内存。
而地址空间这个概念却很少在linux kernel几个经典书籍里面被详细的描述，大多数读者很难体会其中的设计逻辑。

说的经典书籍就是下面几个:

* linux内核设计与实现，英文叫 [linux kernel development](), 作者是[Robert Love](https://rlove.org)，前谷歌工程师，负责安卓系统的底层软件
* linux系统编程，英文叫 [linux system programming]()，作者也是Robert Love。
* 深入理解Linux虚拟内存管理，英文叫 [Understanding the Linux Virtual Memory Manager](https://kernel.org/doc/gorman/)
* linux设备驱动程序，英文叫 [Linux Device Drivers](https://lwn.net/Kernel/LDD3/),由LWN.net知名编辑Jonathan Corbet编写
* 深入理解linux内核，英文叫 [Understanding the Linux Kernel]()

## 引子

最早用汇编写代码的时候，比如8086（x86）上赋值都会用mov指令，比如`mov ax,1234h; mov 1234h, ax`;
C语言，用`int *a; int b; a = &b; *a = 1234;`。
写这些代码的时候，其实我们并没有刻意去想地址空间的的概念，因为我们认为要访问的就是1234h这个物理地址，
或者写的都是由CPU执行的应用程序。

但地址空间首先要分主语，谁的地址空间，是CPU的，还是设备的，
其次也要分软件上看到的虚拟的地址，还是物理的地址，
另外CPU的地址空间可以是一段内存，一个外围设备暴露出来的空间（比如mmio）。

## CPU的虚拟地址

虚拟地址的引入要从两个维度展开，一个是上层应用，一个是体系架构和物理内存的关系。

上层应用的逻辑好理解，如果大家都面向物理地址编程，这个应用很难在其它机器上直接运行。

物理内存的话，其实早期体系架构的空间是大于物理内存的，但是物理内存发展的速度太快，很快物理内存
的容量就超过了体系架构原始定义的空间，为了解决这类问题intel引入了段寄存器，这样原始的地址再加上
段寄存器，就可以映射到不同的物理内存地址了。其实也就是引入了虚拟地址到物理地址的映射。

## MMU的引入

段寄存器之后，intel又加了es扩展段寄存器去支持更大的物理内存，但是ds、es增加了太多的复杂性，
一个专门用于地址翻译的单元MMU(memory management unit）引入了进来，它主要就做一件事，输入一个
虚拟地址，输出一个物理地址，`pa=f(va)`。

这样就需要一个表格，来管理这个映射关系，这个表格也放在物理内存中。如果完全一对一的映射，这个表格就很大，
完全占据了物理内存。

为了裁剪表格的大小，只用一段连续的va映射到一段连续的pa，这样的一段就叫“页”，管理映射的表格就叫“页表”。
一般页的大小和体系架构有关系，默认是4K，在内核的config里面可以设置。

## 进程地址空间

有了虚拟地址，自然多个进程就可以同时运行，但是不同的进程可能都会访问同一个虚拟地址，那怎么映射到不同的
物理地址呢？
多个映射，当然有多个页表，同时体系结上而已有变化，比如arm引入了ASID的概念，每个进程一个ASID，根据ASID
找到各自的页表。

## Linux内核地址空间

大家都知道在linux中，用户态都是通过syscall来做系统调用，内核只单独运行一份。
既然每个进程都有自己的页表，如果进程做系统调用陷入到内核态的话，每个进程有一部分虚拟地址都是映射到
同一片物理地址的。在arm64上，这片内核的虚拟地址通常是`0xFFFF0000_00000000`起头。用高地址映射也是考虑
到进兼容老的代码。


## 设备的地址空间

很早的时候，设备受限于自己寻找空间的限制，只能访问一段低的物理地址，所以给设备分配地址的时候是用
`cpu_va = dma_alloc(dev,size, dma_addr)` 来分配一个设备可以访问的物理地址`dma_addr`。

后来设备做强了，可以把设备访问的有限地址长度转换成和物理空间一样长的地址长度，这个概念类似于CPU侧的
mmu，所以iommu的概念便产生了，arm上也叫smmu。

smmu和mmu一样，也是由多个页表，但是一般来讲smmu并不复用mmu的页表，因为安全原因，我们只希望设备访问到
我们希望设备访问的地址，而不是把所有地址暴露给设备。

有了iommu之后，dma地址也称为iova，也就是设备看到的va。整个逻辑类似`iova=iommu_alloc();iommu_map(domain,iova,pa)`。
具体代码参考:

* [qm驱动代码](https://elixir.bootlin.com/linux/latest/source/drivers/crypto/hisilicon/qm.c#L3100)
* [用户态代码](https://github.com/Linaro/uadk)

## 用户态访问设备

用户态访问设备，如果是设备的寄存器，一般通过`remap_pfn_range`映射给用户空间，因为写寄存器的一般放在控制面，而控制面的
代码基本只有一份。
如果是访问设备的buffer，建议使用`dma_mmap`系函数，因为这组函数会使用设备挂接的iommu的dma函数，如果没有挂iommu，也会
使用`remap_pfn_range`。

## 参考

* [ARMv8-A Address Translation]()
* [地址空间的故事](https://gitee.com/Kenneth-Lee-2012/MySummary/blob/master/%E8%BD%AF%E4%BB%B6%E6%9E%84%E6%9E%B6%E8%AE%BE%E8%AE%A1/%E5%9C%B0%E5%9D%80%E7%A9%BA%E9%97%B4%E7%9A%84%E6%95%85%E4%BA%8B.rst)
* [IOMMU的现状和发展](https://gitee.com/Kenneth-Lee-2012/MySummary/blob/master/%E8%BD%AF%E4%BB%B6%E6%9E%84%E6%9E%B6%E8%AE%BE%E8%AE%A1/IOMMU%E7%9A%84%E7%8E%B0%E7%8A%B6%E5%92%8C%E5%8F%91%E5%B1%95.rst)
* [linux-kernel-memory-map-operations](https://insujang.github.io/2017-04-07/linux-kernel-memory-map-operations/)
