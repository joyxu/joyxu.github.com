---
title: linux device memory
author: Joy Xu
date: 2021-05-25 08:33:08
tags: [linux]
---

## CPU如何访问设备的内存或者寄存器？

一般有以下几种方式：
* pio，port io，和指令集有关系
* mmio，通过ioremap把设备的寄存器或者内存映射到cpu的地址空间中，对这些地址的read和write就相当于操作设备的寄存器了

## 设备如何访问CPU的内存？

设备访问CPU的内存，考虑的事情就比较复杂了，需要考虑设备和cpu、设备和设备之前的同步等问题了。

设备访问cpu的内存一般有以下几种方式，基本上都被linux kernel的DMA api所覆盖：

* 直接访问(dma direct)：设备可以访问的地址和cpu的地址空间刚好重合。
当需要大量连续内存的时候，有两种解决方案：
	* static reserved memory： 系统启动的时候，预留一片内存专门给设备
	* cma：也划分区一片内存给设备使用，有全局的和针对每个设备的，但是和第一个方案不同的是，设备不使用这片内存的时候，系统可以使用。

* 如果地址不重合，则可以借助于iommu，做一个映射
* 如果没有iommu，则需要软件来做映射了，比如通过swiotlb，会在设备无法访问的地址和设备可以访问的地址间做个映射，这个buffer叫bounce buffer。

![device访问ddr](/images/device_access_ddr3.png)

![device访问ddr](/images/device_access_ddr.png)

![device访问ddr](/images/device_access_ddr2.png)

## cma架构

![cma arch](/images/cma-arch.gif)

## swiotl

![swiotlb arch](/images/device_access_swiotlb.png)

## iommu

![iommu arch](/images/device_access_iommu.png)

## 参考

* [IOMMU的现状和发展](https://gitee.com/Kenneth-Lee-2012/MySummary/blob/master/%E8%BD%AF%E4%BB%B6%E6%9E%84%E6%9E%B6%E8%AE%BE%E8%AE%A1/IOMMU%E7%9A%84%E7%8E%B0%E7%8A%B6%E5%92%8C%E5%8F%91%E5%B1%95.rst)
* [Mastering the DMA and IOMMU APIs](https://elinux.org/images/4/49/20140429-dma.pdf)
* [Linux内存管理之CMA](https://www.cnblogs.com/LoyenWang/p/12182594.html)
* [DMA](https://biscuitos.github.io/blog/DMA/)
* [CMA](https://biscuitos.github.io/blog/CMA/)
* [CMA模块学习笔记](http://www.wowotech.net/memory_management/cma.html)
* [Dynamic DMA mapping Guide](http://www.wowotech.net/memory_management/DMA-Mapping-api.html)
* [SWIOTLB 软件IOMMU](https://blog.csdn.net/qq_34719392/article/details/114873284)
* [Linux内存管理：什么是CMA（contiguous memory allocation）连续内存分配器？可与DMA结合使用](https://blog.csdn.net/Rong_Toa/article/details/109558234)
* [没有IOMMU的DMA操作](https://mp.weixin.qq.com/s/wiyLzfnwQAAOn4i7ArgKcA#at)
* [Page migration for IOMMU enhanced devices](https://events.static.linuxfound.org/sites/events/files/slides/main.pdf)
* [DMA -1-（基本）](http://jake.dothome.co.kr/dma-1/)
* [The-Flavors-of-Memory-Supported-by-Linux-their-Use](https://events19.linuxfoundation.org/wp-content/uploads/2017/12/The-Flavors-of-Memory-Supported-by-Linux-their-Use-and-Benefit-Christoph-Lameter-Jump-Trading-LLC.pdf)
* [GUP and ZONE_DEVICE pages](https://lpc.events/event/4/contributions/369/)
* [Minimizing struct page overhead](https://blogs.oracle.com/linux/post/minimizing-struct-page-overhead)
* [DMA CACHE一致性问题解决方案](https://www.cnblogs.com/dream397/p/15660063.html)
