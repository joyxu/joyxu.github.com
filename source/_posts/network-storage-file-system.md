---
title: 支持网络的文件系统底层技术
author: Joy Xu
date: 2022-06-20 02:49:26
tags: [linux, storage, file system, rdma]
---

作为GPU Direct IO技术的延续，这篇讨论下基于网络的文件系统的底层技术。

## 综述

借用一幅图，基本上把所有网络存储技术都包含了进来

![storage network introduction](/images/storage_network.png)

解释下上面涉及到的名词：

* ULP: upper layer protocol，上层协议栈
* libfabric: open fabric（OFED的维护组织）定义的用户态库
* kfabric: open fabric定义的内核模块
* srp: scsi rdma protocl over infiniband
* iser: iscsi extension for rdma
* NVMe/F: NVMe over fabric
* NFSoRDMA: NFS over RDMA
* SMB Direct: server message block, Windows服务器上的技术, MS-SMB
* LNET/LND: Lustre networking，分布式文件系统Lustre的网络技术，其它著名的分布式文件系统还有GlusterFS
* iscsi: scsi based on tcp/ip

## DataStorage 和DataAcces的差异

这里要重点强调下这两个概念，数据的存储和数据的访问是两个不同的概念，
存储一般要经过文件系统，而访问是不用的，由于要经过文件系统，他们的延时差异也很大。

![ds vs da](/images/storage_network_ds_da.png)

![ds da latency](/images/storage_network_ds_da_latency.png)

## libfabric

libfabric一般配合libibverbs(https://github.com/linux-rdma/rdma-core)使用。

![libfabric](/images/storage_network_libfabric.png)

![libfabric](/images/storage_network_libfabric2.png)

但亚马逊的Elastic Fabric Adapter用法不太一样，EFA直接在linux kernel实现了libfabric的provider驱动。

![libfabric](/images/storage_network_libfabric3.png)

## kfabric

在linux kernel，由于上层SRP、iSER、NVMe/F、NFSoRDMA要调用ib的verbs，而各个厂家的驱动层也要实现
类似的回调函数，OFED组织又做了一层抽象框架命名为kfabric，但是这个模块还一直没有何如到linux kernel主线，
最近的一次更新也是在16年了，可以认为已经停止开发了。

![kfabric](/images/storage_network_kfabric.png)

主要用于用LNET文件系统，可惜从linux kernel 4.18之后，lustre分布式文件系统被移除了内核，
但lustre官方仍在维护out of tree的代码，且由于ARM在hpc领域的绽放，linaro也在维护arm的版本，具体可以参考他们的官网。

![kfabric](/images/storage_network_kfabric2.png)

## libfabric和kfabric

根据上节的介绍，其实libfabric和kfabric并不一定要配合使用，具体差异参考下图

![libfabric vs kfabric](/images/storage_network_fabric.png)

## SRP & ISER

SRP和ISER都是已经在内核支持的协议，只需要把相应config选项打开就可以使能了。

![srp kernel configure](/images/storage_network_srp.png)

![iser kernel configure](/images/storage_network_iser.png)

相对来讲，ISER比SRP更好，具体可以从以下几个方面对比：

![srp & iser](/images/storage_network_srp_iser3.png)

iser一般组网的方式如下：

![iser case](/images/storage_network_iser_case.png)

iser的调用栈如下

![srp & iser](/images/storage_network_srp_iser.png)

![srp & iser](/images/storage_network_srp_iser2.png)

![iser e2e](/images/storage_network_iser_e2e.png)

关键代码流程：

	Initiator:
	iscsi_data_xmit
	  xmit_task
	    iser_send_control(registered completed callback iser_ctrl_comp)
	      ib_dma_sync_xxx


	Target:


数据流图如下

![iser dataflow](/images/storage_network_iser_dataflow.png)

以mlx为例，kernel中的关系图如下

![iser kernel](/images/storage_network_iser_kernel.png)

2.6的kernel中，storage的模块关系如下

![iser kernel](/images/storage_network_26linux.png)

## NVMe/F

## SMB

由于是windows上的技术，这里就不介绍了。

## 八卦

以前高端网卡基本都是intel和broadcom的市场，后面微软找MLX在数据中心合作，结果
MLX在25G时代一把上位。后面亚马逊又收购了同处于以色列的Annapuna Lab，做了自己的
RDMA，也就是后面的EFA(https://github.com/amzn/amzn-drivers)。

## 参考

* [Storage Networking Industry Association](https://www.snia.org/)
* [Persistent Memory over Fabrics An Application-centric view](https://www.snia.org/sites/default/files/PM-Summit/2017/presentations/Paul_Grun_Doug_Voigt_PM_over_Fabrics-an-Application-centered_Viewv2.pdf)
* [Persistent Memory over Fabric Architecture Overview](https://slideplayer.com/slide/13896886)
* [Set up Message Passing Interface for HPC](https://docs.microsoft.com/en-us/azure/virtual-machines/workloads/hpc/setup-mpi)
* [Now Available – Elastic Fabric Adapter (EFA) for Tightly-Coupled HPC Workloads](https://aws.amazon.com/cn/blogs/aws/now-available-elastic-fabric-adapter-efa-for-tightly-coupled-hpc-workloads/)
* [Pathfinding a Kernel Storage Fabric Mid-layer](https://openfabrics.org/images/eventpresos/2016presentations/106kfabric.pdf)
* [云计算三大神器来了！CPU、GPU、DPU](https://www.sohu.com/a/427490498_505795)
* [深入理解Lustre文件系统-第3篇 LNET：Lustre网络](https://blog.csdn.net/fsdev/article/details/7800126)
* [lustre patchwork](https://patchwork.kernel.org/project/lustre-devel/list/)
* [A triad-based architecture for a multipurpose Lustre filesystem at /rdlab](https://rdlab.cs.upc.edu/wp-content/uploads/2021/05/LUG2021-Triad-based-architecture-rdlab.pdf)
* [NFS over SoftRoCE setup](http://linux-nfs.org/wiki/index.php/NFS_over_SoftRoCE_setup)
* [NFS/RDMA README](https://kernel.org/doc/Documentation/filesystems/nfs/nfs-rdma.txt)
* [NFS/RDMA Next Steps](https://datatracker.ietf.org/meeting/99/materials/slides-99-nfsv4-nfsrdma-next-steps-chuck-lever-00)
* [RDMA in Data Centers: Looking Back and Looking Forward](https://slidetodoc.com/rdma-in-data-centers-looking-back-and-looking/)
* [The role of a InfiniBand and automated data tiering in achieving extreme storage performance](https://kipdf.com/the-role-of-a-infiniband-and-automated-data-tiering-in-achieving-extreme-storage_5aef9bb47f8b9a0f648b4583.html)
* [NVIDIA MLNX_OFED Documentation Rev 5.3-1.0.5.0 Introduction](https://docs.nvidia.com/networking/display/MLNXOFEDv531050/Introduction)
* [iSER as accelerator for Software Defined Storage](https://www.snia.org/sites/default/files/SDC/2016/presentations/storage_networking/RahulFiske_iSER_Accelerator_Software_Defined_Storage_v2.pdf)
* [What is ISER?](https://support.mellanox.com/s/article/what-is-iser-x)
* [Mellanox Linux Driver Modules Relationship](https://support.mellanox.com/s/article/mellanox-linux-driver-modules-relationship--mlnx-ofed-x)
* [The OFED package](https://www.rdmamojo.com/2012/04/25/the-ofed-package/)
* [Linux/iSCSI and a Generic Target Mode Storage Engine for Linux v2.6](https://www.usenix.org/legacy/events/lsf08/tech/IO_bellinger.pdf)
