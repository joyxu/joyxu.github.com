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

由于bar是作为`zone_device`注册的,`zone_device`仍然是基于`sparsemem_vmemmap`的，它提供了基于`struct page`的服务，包括`pfn_to_page`,`page_to_pfn`和`get_user_pages`等。
但它的分配和释放并不是当做普通内存来管理的，而是通过generic purpose allocator来管理分配的，并会针对该设备创建一个内存池。
当map的时候，会调用`p2pmem_alloc_mmap`，并通过`vm_insert_page`把这个地址映射到用户态的va。

* 对端设备怎么用这个地址呢？

另外一个设备拿到这个地址之后，怎么找到这个地址对应的设备的bar offset呢？
这就有回到类似dma的作用，把dma地址映射到内存的物理地址，而在p2p场景中，则是把dma地址，转换成bar offset。
要理解这个细节，可以看`pci_p2pmem_virt_to_bus`，它会调用`gen_pool_virt_to_phys`，返回pcie bar的offset，于是对端设备实际上配置的是pcie的地址空间。
于是当对端设备针对这个地址发起访问的时候，就没有上pcie root port，而是直接到该设备了。

注意：
当前内核一旦开启了P2P，默认会关掉ACS，也可能会影响到SVA。
原则上ATS/ACS和P2P是有些冲突的，因为当ATS打开之后，设备发出的PCIe TLP报文会声称该报文的地址是否是翻译过的。
如果没有翻译，则先路由到RC的TA处进行地址翻译;如果翻译过，则直接使用，绕过了IOMMU的隔离，直接访问这个物理地址了，导致安全风险。
比如说，开启了P2P和ATS以后，同一个PCIe Switch后的所有EP设备，必须都分给同一个虚拟机，不然分给不同虚拟机的话，可以从这个PCIe设备的另外一个Function攻击到其它的虚拟机。
于是呢，就引入了ACS（访问控制）来决定一个TLP是否能正常路由，还是被阻塞或者重定向。
可以参考这个patch的评论[PCI/P2PDMA: Clear ACS P2P flags for all devices behind switches](https://patchwork.kernel.org/project/linux-pci/patch/20180312193525.2855-5-logang@deltatee.com/)。
也可以参考SBSA的测试用例[SBSA PCIe ATS test](https://github.com/ARM-software/sbsa-acs/issues/111)。

# 其它

PCIe TLP报文格式

![pcie tlp](/images/pcie_tlp.png)

ATS域段

![pcie tlp_ats](/images/pcie_tlp_ats.png)

![pcie_ats](/images/pcie_ats.png)

# 参考

* [PCIe扫盲系列博文连载目录篇](http://blog.chinaaet.com/justlxy/p/5100053328)
* [PCIe扫盲——PCI总线的三种传输模式](http://blog.chinaaet.com/justlxy/p/5100053095)
* [PCIe ECAM机制](https://blog.csdn.net/u013253075/article/details/130755162)
* [Linux源码阅读——PCI总线驱动代码（三）PCI设备枚举过程](https://blog.csdn.net/u013253075/article/details/123301127)
* [Writing a PCI device driver for Linux](https://olegkutkov.me/2021/01/07/writing-a-pci-device-driver-for-linux/)
* [How To Write Linux PCI Drivers](https://www.kernel.org/doc/html/latest/PCI/pci.html)
* [A Practical Tutorial on PCIe for Total Beginners on Windows (Part 1)](https://ctf.re/windows/kernel/pcie/tutorial/2023/02/14/pcie-part-1/)
* [UEFI——PCIe子系统(I)](https://blog.csdn.net/weixin_43921686/article/details/132136732)
* [Introduction to PCIe Address Translation Services](https://liujunming.top/2019/11/24/Introduction-to-PCIe-Address-Translation-Services/)
* [PCIe TLP Header 中的常见 Feild 及其释义](https://mangopapa.blog.csdn.net/article/details/128538065)
* [PCIe地址转换服务（ATS）详解](https://mangopapa.blog.csdn.net/article/details/120245027)
* [PCIe地址转换服务（ATS）详解](https://github.com/yakoye/PCIeDocs/blob/main/PCIe%E5%9C%B0%E5%9D%80%E8%BD%AC%E6%8D%A2%E6%9C%8D%E5%8A%A1%EF%BC%88ATS%EF%BC%89%E8%AF%A6%E8%A7%A3.md)
* [PCIe访问控制服务（ACS）](https://mangopapa.blog.csdn.net/article/details/120295827)
* [PCI设备驱动（二）](https://blog.csdn.net/21cnbao/article/details/105525581)
* [P2P DMA](https://zhuanlan.zhihu.com/p/664873131)
* [PCI Peer-to-Peer DMA Support](https://www.kernel.org/doc/html/next/driver-api/pci/p2pdma.html)
* [Peer-to-peer DMA](https://lwn.net/Articles/931668/)
* [PCIe扫盲——一个Memory Read操作的例子](http://blog.chinaaet.com/justlxy/p/5100053263)
* [Down to the TLP: How PCI express devices talk](https://xillybus.com/tutorials/pci-express-tlp-pcie-primer-tutorial-guide-1)
* [颠覆性技术！你NRZ相守20年又怎样？看我PAM4如何上位PCIe 6.0](https://mangopapa.blog.csdn.net/article/details/120775889)
* [p2pmem-test](https://github.com/sbates130272/p2pmem-test)
* [Linux SVA特性分析](https://blog.csdn.net/scarecrow_byr/article/details/100983619?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522170263012016800180699660%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=170263012016800180699660&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_ecpm_v1~rank_v31_ecpm-1-100983619-null-null.nonecase&utm_term=sva&spm=1018.2226.3001.4450)
* [How can my PCI device driver remap PCI memory to userspace](https://stackoverflow.com/questions/66893486/how-can-my-pci-device-driver-remap-pci-memory-to-userspace)
