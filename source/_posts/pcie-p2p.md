---
title: pcie、p2p和ATS分析
author: Joy Xu
date: 2023-12-11 02:15:15
tags: [linux kernel, pcie, p2p]
---

# 综述

PCIe协议定义了PCIe设备三种数据传输方式之一(PIO，P2P和DMA)，分别对应到CPU访问设备，设备访问设备和设备访问内存/CPU。

![pcie data_transfer](/images/pcie_data_transfer.png)

# CPU访问设备-PCIe设备枚举建链

## PCI设备的地址空间

PCI协议定义了三种地址空间：mmio地址空间(memory address space)，io地址空间(io address space)和配置空间(configure address space)。
CPU要访问前面两种地址空间分别通过mmio和pio，而对于配置空间则使用io端口 CF8/CFC或者ECAM方式访问，在arm64平台上只支持ECAM机制。

## ECAM机制

ECAM其实也是基于mmio访问的，以ACPI启动举例，ACPI的MCFG(memory-maped configuration)表格中会配置ECAM的基地址。

![pcie ecam](/images/pcie_ecam_mcfg.png)

完整流程如下：

![pcie ecam_overview](/images/pcie_ecam_overview.png)

## 设备枚举建链接

建立好配置空间之后，后面就是读取配置空间信息，建立pcie设备树，具体流程如下:

	acpi_init
	  acpi_scan_init
	    *acpi_pci_root_init
	      acpi_pci_root_add
	        pci_acpi_scan_root
		  acpi_pci_root_create
		    pci_create_root_bus
		    pci_scan_child_bus
		      pci_scan_single_device
		        pci_scan_device
			  pci_setup_device
		        pci_device_add

	    *acpi_pci_link_init

其中`acpi_pci_root_init`完成pci设备的相关操作，包括设备枚举、配置空间设置等，最终完成物理设备到逻辑设备的映射：

![pcie ecam_scan](/images/pcie_ecam_scan.png)

# 设备访问内存、CPU

这块涉及到的技术主要包括IOMMU，SVA，DDIO等，就不在本文描述了。

# 设备访问设备（P2P）

PCIe P2P是指两个PCIe设备直接通信，通信的数据不经过CPU处理从一个PCIe设备交换到另外一个PCIe设备。
因此有两个隐藏的前提条件：

* 至少某一个PCIe设备拥有内存，而不是简单的一个PCIe设备去访问另外一个PCIe设备的BAR空间或者寄存器
* 至少某一个PCIe设备拥有算力

前面已经写过一篇[P2P的文章](https://joyxu.github.io/2022/07/19/p2p-dma%E6%8A%80%E6%9C%AF%E5%88%86%E6%9E%90%E6%80%BB%E7%BB%93/)，
现在6.2内核已经把`p2pdma`接口合并到`dma`接口，并通过vfio接口export到了用户态。
简单来讲，主要实现以下2点：

* 写一个pcie的设备驱动，实现p2pdma provider的`pci_p2pdma_add_resource`接口，将bar作为`zone_device`注册成为DMA内存资源，
并通过`pci_p2pm_publish`发布这个资源，允许p2pdma的orchestrator调用。注意如果该设备已有驱动匹配了，要先unbind原有驱动。

* 用户态通过`mmap`和`munmap`从这个设备以类似`O_Direct`方式来分配和释放内存，并在用户态通过SPDK类似的软件，在另外一个设备中操作该地址，或者直接在用户态读写该内存。

## 原理

* mmap怎么从这个bar分配内存呢？

由于bar是作为`zone_device`注册的，它的分配和释放并不是通过sparse来管理的，而是通过generic purpose allocator来管理分配的，并会针对该设备创建一个内存池。
当map的时候，会调用`p2pmem_alloc_mmap`，并通过`vm_insert_page`把这个地址映射到用户态的va。

* 对端设备怎么用这个地址呢？

另外一个设备拿到这个地址之后，怎么找到这个地址对应的设备的bar offset呢？
这就有回到类似dma的作用，把dma地址映射到内存的物理地址，而在p2p场景中，则是把dma地址，转换成bar offset。
要理解这个细节，可以看`pci_p2pmem_virt_to_bus`，它会调用`gen_pool_virt_to_phys`，返回pcie bar的offset，于是对端设备实际上配置的是pcie的地址空间。
于是当对端设备针对这个地址发起访问的时候，就没有上pcie root port，而是直接到该设备了。

注意，原则上ATS和P2P是冲突的，因为当ATS打开之后，设备不清楚拿到的地址是否还要走ATS转换。

# 其它

PCIe TLP报文格式

![pcie tlp](/images/pcie_tlp.png)

# 参考

* [PCIe扫盲系列博文连载目录篇](http://blog.chinaaet.com/justlxy/p/5100053328)
* [PCIe扫盲——PCI总线的三种传输模式](http://blog.chinaaet.com/justlxy/p/5100053095)
* [PCIe ECAM机制](https://blog.csdn.net/u013253075/article/details/130755162)
* [Linux源码阅读——PCI总线驱动代码（三）PCI设备枚举过程](https://blog.csdn.net/u013253075/article/details/123301127)
* [A Practical Tutorial on PCIe for Total Beginners on Windows (Part 1)](https://ctf.re/windows/kernel/pcie/tutorial/2023/02/14/pcie-part-1/)
* [UEFI——PCIe子系统(I)](https://blog.csdn.net/weixin_43921686/article/details/132136732)
* [Introduction to PCIe Address Translation Services](https://liujunming.top/2019/11/24/Introduction-to-PCIe-Address-Translation-Services/)
* [PCIe地址转换服务（ATS）详解](https://github.com/yakoye/PCIeDocs/blob/main/PCIe%E5%9C%B0%E5%9D%80%E8%BD%AC%E6%8D%A2%E6%9C%8D%E5%8A%A1%EF%BC%88ATS%EF%BC%89%E8%AF%A6%E8%A7%A3.md)
* [PCI设备驱动（二）](https://blog.csdn.net/21cnbao/article/details/105525581)
* [P2P DMA](https://zhuanlan.zhihu.com/p/664873131)
* [PCI Peer-to-Peer DMA Support](https://www.kernel.org/doc/html/next/driver-api/pci/p2pdma.html)
* [Peer-to-peer DMA](https://lwn.net/Articles/931668/)
* [PCIe扫盲——一个Memory Read操作的例子](http://blog.chinaaet.com/justlxy/p/5100053263)
