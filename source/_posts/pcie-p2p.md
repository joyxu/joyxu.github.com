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

其中`acpi_pci_root_init`完成pci设备的相关操作，包括设备枚举、配置空间设置等。

# 设备访问内存、CPU

这块涉及到的技术主要包括IOMMU，SVA，DDIO等，就不在本文描述了。

# 设备访问设备（P2P）

PCIe P2P是指两个PCIe设备直接通信，通信的数据不经过CPU处理从一个PCIe设备交换到另外一个PCIe设备。
因此有两个隐藏的前提条件：

* 至少某一个PCIe设备拥有内存，而不是简单的一个PCIe设备去访问另外一个PCIe设备的BAR空间或者寄存器
* 至少某一个PCIe设备拥有算力




# 参考

* [PCIe扫盲系列博文连载目录篇](http://blog.chinaaet.com/justlxy/p/5100053328)
* [PCIe扫盲——PCI总线的三种传输模式](http://blog.chinaaet.com/justlxy/p/5100053095)
* [PCIe ECAM机制](https://blog.csdn.net/u013253075/article/details/130755162)
* [Linux源码阅读——PCI总线驱动代码（三）PCI设备枚举过程](https://blog.csdn.net/u013253075/article/details/123301127)
* [A Practical Tutorial on PCIe for Total Beginners on Windows (Part 1)](https://ctf.re/windows/kernel/pcie/tutorial/2023/02/14/pcie-part-1/)
* [UEFI——PCIe子系统(I)](https://blog.csdn.net/weixin_43921686/article/details/132136732)
* [Introduction to PCIe Address Translation Services](https://liujunming.top/2019/11/24/Introduction-to-PCIe-Address-Translation-Services/)
* [PCIe地址转换服务（ATS）详解](https://github.com/yakoye/PCIeDocs/blob/main/PCIe%E5%9C%B0%E5%9D%80%E8%BD%AC%E6%8D%A2%E6%9C%8D%E5%8A%A1%EF%BC%88ATS%EF%BC%89%E8%AF%A6%E8%A7%A3.md)
