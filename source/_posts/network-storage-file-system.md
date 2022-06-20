---
title: 支持网络的文件系统底层技术
author: Joy Xu
date: 2022-06-20 02:49:26
tags: [linux, storage, file system, rdma]
---

作为GPU Direct IO技术的延续，这篇讨论下基于网络的文件系统的底层技术。

## 综述

借用一幅图，基本上把所有网络存储技术都包含了进来

![storage network introduction](/imags/storage_network.png)

解释下上面涉及到的名词：

* ULP: upper layer protocol，上层协议栈
* libfabric: open fabric（OFED的维护组织）定义的用户态库
* kfabric: open fabric定义的内核模块
* srp: scsi rdma protocl over infiniband
* iser: iscsi extension for rdma
* NVMe/F: NVMe over fabric
* NFSoRDMA: NFS over RDMA
* SMB Direct: server message block, Windows服务器上的技术, MS-SMB
* LNET/LND: Lustre networking，分布式文件系统Lustre的网络技术
* iscsi: scsi based on tcp/ip

## DataStorage 和DataAcces的差异

这里要重点强调下这两个概念，数据的存储和数据的访问是两个不同的概念，
存储一般要经过文件系统，而访问是不用的，由于要经过文件系统，他们的延时差异也很大。

![ds vs da](/imags/storage_network_ds_da.png)

![ds da latency](/imags/storage_network_ds_da_latency.png)

## libfabric

libfabric一般配合libibverbs(https://github.com/linux-rdma/rdma-core)使用。

![libfabric](/imags/storage_network_libfabric.png)

![libfabric](/imags/storage_network_libfabric2.png)

但亚马逊的Elastic Fabric Adapter用法不太一样，EFA直接在linux kernel实现了libfabric的provider驱动。

![libfabric](/imags/storage_network_libfabric3.png)

## kfabric

在linux kernel，由于上层SRP、iSER、NVMe/F、NFSoRDMA要调用ib的verbs，而各个厂家的驱动层也要实现
类似的回调函数，OFED组织又做了一层抽象框架命名为kfabric，但是这个模块还一直没有何如到linux kernel主线，
最近的一次更新也是在16年了，可以认为已经停止开发了。

![kfabric](/imags/storage_network_kfabric.png)

主要用于用LNET文件系统，可惜从linux kernel 4.18之后，lustre分布式文件系统被移除了内核，
但lustre官方仍在维护out of tree的代码，且由于ARM在hpc领域的绽放，linaro也在维护arm的版本，具体可以参考他们的官网。

![kfabric](/imags/storage_network_kfabric2.png)

## libfabric和kfabric

根据上节的介绍，其实libfabric和kfabric并不一定要配合使用，具体差异参考下图

![libfabric vs kfabric](/imags/storage_network_fabric.png)


## 参考

* [Storage Networking Industry Association](https://www.snia.org/)
* [Persistent Memory over Fabrics An Application-centric view](https://www.snia.org/sites/default/files/PM-Summit/2017/presentations/Paul_Grun_Doug_Voigt_PM_over_Fabrics-an-Application-centered_Viewv2.pdf)
* [Persistent Memory over Fabric Architecture Overview](https://slideplayer.com/slide/13896886)
* [Set up Message Passing Interface for HPC](https://docs.microsoft.com/en-us/azure/virtual-machines/workloads/hpc/setup-mpi)
* [Now Available – Elastic Fabric Adapter (EFA) for Tightly-Coupled HPC Workloads](https://aws.amazon.com/cn/blogs/aws/now-available-elastic-fabric-adapter-efa-for-tightly-coupled-hpc-workloads/)
* [Pathfinding a Kernel Storage Fabric Mid-layer](https://openfabrics.org/images/eventpresos/2016presentations/106kfabric.pdf)
* [云计算三大神器来了！CPU、GPU、DPU](https://www.sohu.com/a/427490498_505795)
