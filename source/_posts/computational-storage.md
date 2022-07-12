---
title: computational storage计算存储
author: Joy Xu
date: 2022-07-12 03:46:28
tags: [linux, storage, compute, p2pdma]
---

感觉这年头干啥芯片的都想把计算绑定在一起，之前有网卡把计算绑一起，现在也有把存储和计算绑一起。

## 什么是计算存储

其实这要从NVME说起，从NVME 1.2规范引入CMB(Controller Memory Buffer)这个概念起，就慢慢有了变革的火种。
CMB是一段放在NVME卡上的内存，有点类似于独显的显存的概念，也是通过PCIe BAR空间对CPU可见，一开始主要是
把NVME涉及到的SQ和CQ放在这个内存区域，避免访问系统DDR，而起到减小延迟的效果。

那现在有了内存，为什么不参考下GPU，把计算core放进去呢，这样就有了computational storage的概念。
借用下SNIA组织的图，更形象的说明下什么是computational storage：

![compute storage][/images/compute_storage1.png]

## 代表公司和产品

推动computational storage的主要有三家初创公司: Eideticom, NGD和scaleflux。
当然还有传统的nvme存储公司比如美光和三星，这几家里面我认为软件生态做的最好的是Eideticom。
 
为了有更直观的感受，放个Eideticom卡的图

![compute storage][/images/compute_storage_card.png]

典型的用户场景是通过减少数据移动，把一些原先要在cpu和nvme路径上执行的任务，直接卸载到computational storage processor里面来，提升效率。比如：

* 数据库场景中数据的压缩计算
* 网络数据的存储，这个有点类似于nvme over rdma，且把计算也offload到nvme卡上，具体示意如下:

![compute storage][/images/compute_storage3.png]

## p2pdma

为了让NMVE卡上的这块内存可以被使用，就不得不提p2pdma这个上了linux和qemu主线的方案，
前面也介绍过mlx 的peer to peer direct技术，也介绍过nvidia的 gpudirect技术，但是它们目前都没上linux主线。

p2pdma在内核的pci驱动目录中的p2pdma.c中，从4.20开始被支持，想了解历史的看看这组patch和相关作者的一个总结:

* [Copy Offload in NVMe Fabrics with P2P PCI Memory](https://patchwork.kernel.org/project/linux-nvdimm/cover/20181004212747.6301-1-logang@deltatee.com/)
* [p2pdma in Linux kernel 4.20-rc1 is here!](https://www.eideticom.com/media-news/blog/33-p2pdma-in-linux-kernel-4-20-rc1-is-here.html)
* [Device-to-device memory-transfer offload with P2PDMA](https://lwn.net/Articles/767281/)

在这个总结中，作者有提到在arm64上还需要加上ioremap相关的patch才能正常工作，不知道现状如何了。

卡上的内存暴露出来之后，下一步就是怎么把这块内存给用起来。
当前能直接使用p2pdma技术的主要有SPDK，以及Eideticom自家的libnoload。

![compute storage][/images/compute_storage2.png]

## SPDK

这里以SPDK为例，
感兴趣的可以直接看spdk中的 [cmb_copy.c](https://github.com/spdk/spdk/blob/master/examples/nvme/cmb_copy/cmb_copy.c#L80)，感觉用起来还是很简单的。

	spdk_nvme_ctrlr_map_cmb
	 nvme_transport_ctrlr_map_cmb
	  spdk_nvme_transport_ops.ctrlr_map_cmb
	   nvme_pcie_ctrlr_map_io_cmb
	    nvme_pcie_ctrlr_get_cmbsz
	     spdk_mmio_read
	    根据暴露出来的bar_va, 计算buf
	    spdk_mem_register(buf)

	spdk_nvme_ns_cmd_read
	 nvme_qpair_submit_request
	  nvme_transport_qpair_submit_request
	   spdk_nvme_transport_ops.qpair_submit_request
	    nvme_pcie_qpair_submit_request
	     nvme_pcie_qpair_submit_tracker
	      nvme_pcie_qpair_ring_sq_doorbell
	       nvme_pcie_qpair_update_mmio_required

## 把算力用起来

要把算力用起来，就要借助一个其它的PCIe 设备，比如网卡，可以参考下面这个仓库，看看怎么用起来的，主要是把网卡的收到的NVMe包，直接放到NVME卡的CMB中：

* [Offload+p2pdma kernel code](https://github.com/lsgunth/linux/tree/max-mlnx-offload-p2pdma)

## 参考

* [What Is Computational Storage?](https://www.snia.org/education/what-is-computational-storage)
* [NVME CMB详解](https://zhuanlan.zhihu.com/p/457874205)
* [Accelerating Storage with NVM Express SSDs and P2PDMA](https://www.snia.org/sites/default/files/SDC/2018/presentations/Storage_Architecture/Bates_Stephen_Accelerating_Storage_with_NVM_Express_SSDs_and_P2PDMA.pdf)
* [p2pdma__why_how_what](https://lpc.events/event/2/contributions/136/attachments/164/379/p2pdma__why_how_what_.pdf)
* [p2pmem github code repo](https://github.com/sbates130272/linux-p2pmem)
* [NoLoad U.2 Computational Storage Processor](https://www.eideticom.com/uploads/attachments/2019/07/31/noload_csp_u2_product_brief.pdf)
* [Enabling the NVMe™ CMB and PMR Ecosystem](https://nvmexpress.org/wp-content/uploads/Session-2-Enabling-the-NVMe-CMB-and-PMR-Ecosystem-Eideticom-and-Mell....pdf)
