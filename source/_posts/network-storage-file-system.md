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

本文只介绍数据存储，接下来围绕开篇提到的几个技术展开。

## DAS 直连式存储

讲网络存储之前，还是先介绍下传统的Direct Attached Storage直连式存储。
说白了，就是常见的PC或者服务器直接连着硬盘的。
以常见的scsi为例子，应用下发的文件访问，在scsi层都转换为scsi命令发给scsi存储控制器处理，
对应到kernel中，就是常见的 [scsi驱动](https://elixir.bootlin.com/linux/v5.18.8/source/drivers/scsi/mpt3sas) 了。

如果对整个层次不清楚的话，可以了解下linux storage的全景图：

![linux storage stack overview](/images/Linux-storage-stack-diagram_v4.10.svg)

## ISCSI

为了支持网络存储，linux kernel在scsi层下加了一个传输层，把scsi命令通过网络报文发出来。
这个发出报文的client,在scsi命名空间中称为initiator, 而接受处理报文的称为target，这两个名词概念后文会税涉及到。

![iscsi packet](/images/storage_network_iscsi_packet.png)

### SRP & ISER

SRP和ISER均能通过RDMA支持SCSI，且都已经被内核支持。
相对来讲，ISER比SRP更好，具体可以从以下几个方面对比：

![srp & iser](/images/storage_network_srp_iser3.png)

iser典型应用场景如下：

![iser case](/images/storage_network_iser_case.png)

其中的数据流如下：

![iser dataflow](/images/storage_network_iser_dataflow.png)

以mlx为例，kernel中的逻辑视图如下：

![iser kernel](/images/storage_network_iser_kernel.png)

### ISER Initiator

内核中iser intiator位于infiniband/ulp中，它会注册到scsi transport里面，把scsi报文封装到IB报文中。

![iser kernel configure](/images/storage_network_iser_initiator.png)

调用栈如下（不包括scsi层之上）

	scsi_transport_iscsi.c
		netlink_kernel_create(&iscsi_if_rx)
		iscsi_if_rx
		 iscsi_if_recv_msg
		  transport->create_session
		  transport->create_conn
		  transport->bind_conn
		  transport->create_session
		  transport->create_session

	ib_isert.ko:
		iser_init
		 iscsi_register_transport(&iscsi_iser_transport)

	scsi_lib.c:
		scsi_dispatch_cmd
		 transport::queuecommand
		  iscsi_queuecommand
		   iscsi_data_xmit
		    xmit_task
		     iser_send_control(registered completed callback iser_ctrl_comp)
		      ib_dma_sync_xxx

### ISER Target

Target侧的实现有很多种方案，主流主要有三种：

* LIO/TCMU: LIO后面名字改成了TCM(Target Core Mod), TCMU是TCM的用户态，已经被Linux Kernel主线支持。由linux-iscsi.org运营维护,主要的开发者是Nicholas A.Bellinger。
* [SCST](http://scst.sourceforge.net/index.html): the generic SCSI target subsystem for Linux,名字最官方，可惜一直没合入到Linux社区，且和LIO的作者一直不合。
* [STGT(也叫TGT)](https://github.com/fujita/tgt): 作者是NTT的FUJITA Tomonori和Redhat的Mike Christie，最早的版本是包含内核态的，后面去掉了。

推荐使用LIO/TCMU方案。
其中STGT是纯用户态的，但基本不用了，SCST和LIO/TCMU都有内核模块，但SCST的内核模块并没有合入到内核主线。
更详细的对比可以参考 [SCST的官网](http://scst.sourceforge.net/comparison.html)。

![iser targets](/images/storage_network_iser_targets.png)

#### STGT(TGT)

网上的材料都比较老了，很多图里面都画着它有内核态，最初的版本确实是有，只是后面都去掉了，
这个链接的patch是最早在kernel中加入netlink的patch：[scsi tgt: scsi target netlink interface](https://lwn.net/Articles/172694/)
喜欢考古的兄弟可以结合github上老的tag版本结合一起看看。

最新的版本是纯用户态的，iser的支持都在user/iscsi/iser.c中，简单的调用流如下:

	iser.c:
	 iser_device_init
	  ibv_create_cq
	  ibv_req_notify_cq
	  tgt_event_add(cq->fd, &iser_handle_cq_event)

	 iser_handle_cq_event(如果ib有cq产生，fd唤醒epoll)
	  iser_poll_cq(ibv_poll_cq)
	   handle_wc
	   iser_queue_task
	    iser_sched_iosubmit
	     iser_scsi_cmd_iosubmit
	      target_cmd_queue
	      	cmd_perform
		 sbc_rw
		  bs_cmd_submit
		   bs_rdwr_request
		    pread/pwrite(系统调用，读写)
		  
	tgtd.c:
	 main
	  event_loop
	   epoll_wait(cq->fd)
	    handler

iser的流程图就不画了，和下面这个tcp的图很类似：

![iser tgt tcp](/images/storage_network_iser_tgt_tcp.png)

#### SCST

源码除了从官网下载，也可以从github下载 [SCST-project](https://github.com/SCST-project/scst), 其实还是很活跃的，
从2.6到5.14的内核都可以支持，最新的tag版本是v3.6，发布与2022年1月。

SCST的作者和LIO的作者似乎有些不同的意见，在其官网中也一直强调它的优势，由于精力实在有限，就不深入挖掘它的用法了。
David画了一个很漂亮的图:

![iser SCST](/images/storage_network_iscsi.png)

#### LIO/TCMU

有看到些信息说中国移动使用了这个方案。

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


## NVMe/F

## SMB

由于是windows上的技术，这里就不介绍了。

## 八卦

以前高端网卡基本都是intel和broadcom的市场，后面微软找MLX在数据中心合作，结果
MLX在25G时代一把上位。后面亚马逊又收购了同处于以色列的Annapuna Lab，做了自己的
RDMA，也就是后面的EFA(https://github.com/amzn/amzn-drivers)。

## 参考

* [Storage Networking Industry Association](https://www.snia.org/)
* [Evolution of iSCSI](https://www.snia.org/sites/default/files/ESF/Evolution_of_iSCSI_Final.pdf)
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
* [Performance Implications Libiscsi RDMA support](https://www.snia.org/sites/default/files/SDC/2016/presentations/storage_networking/Shterman-Grimberg_Greenberg_Performance%20Implications%20Libiscsi_%20RDMA_V6.pdf)
* [How Ethernet RDMA Protocols iWARP and RoCE Support NVMe over Fabrics](https://www.snia.org/sites/default/files/ESF/How_Ethernet_RDMA_Protocols_Support_NVMe_over_Fabrics_Final.pdf)
* [Ethernet Storage Fabrics: Using RDMA with Fast NVMe-oF Storage to Reduce latency and Improve Efficiency](https://www.snia.org/educational-library/ethernet-storage-fabrics-using-rdma-fast-nvme-storage-reduce-latency-and-improve)
* [Linux iSCSI](https://runsisi.com/2019/10/30/linux-iscsi/)
* [SCST-Usermode-Adaptation](https://davidbutterfield.github.io/SCST-Usermode-Adaptation/)
* [Storage Stack](https://wxdublin.gitbooks.io/deep-into-linux-and-beyond/content/io.html)
* [Linux Storage Stack Diagram](https://www.thomas-krenn.com/en/wiki/Linux_Storage_Stack_Diagram)
* [Linux LIO 与 TCMU 用户空间透传](http://bos.itdks.com/6267b2df606e482085e35322c7fae55b.pdf)
* [tgt服务端流程分析](https://blog.csdn.net/tdaajames/article/details/80983309?spm=1001.2014.3001.5502)
* [tgt作者 FUJITA Tomonori个人主页](https://bitset.dev/)
* [Linux中三种SCSI target的介绍之各个target的优劣](https://blog.csdn.net/scaleqiao/article/details/46761993)
