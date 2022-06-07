---
title: gpu-direct
author: Joy Xu
date: 2022-06-06 02:15:32
tags: [linux, gpu, rdma]
---

## GPUDirect

GPUDirect并不是一门很新的技术了，这个概念由Nvidia在2012年Kepler这一代GPU微架构率先提出来。
结合RDMA技术，它允许单机或者网络中的GPU可以互相交换数据，而不用经过CPU/系统内存。
结合PCIe的Peer-to-Peer技术，它可以直接访问同一个PCIe rootport下的其它设备，而不用绕一圈到cpu或者系统内存。
更形象的看，可以看看下面两个图，做个对比，就更加直观了：

![without gpudirect](/images/without_gpudirect.png)

![with_gpudirect](/images/with_gpudirect.png)

再深入之前，先了解下P2P技术的几个阶段：

![p2p history](/images/p2p_history.png)

![p2p history2](/images/p2p_history2.png)

有了P2P之后，作为设备厂商的Nvidia一直想着怎么旁路CPU，抢占intel的市场，推出了各种direct技术。
最后发展成nvidia magnum io。后文会围绕这些技术展开。

![p2p history3](/images/p2p_history3.png)

## 地址的问题

GPUDirect要解决的第一个问题，是地址的问题，要保证设备之间地址的一致性，或者说要让其它设备认识GPU的地址。

在linux kernel中，普通的设备驱动的大致流程，都是用户态把数据读/写到用户态VA，内核把VA转换成PA，
驱动程序在把PA转换成IOVA（或者说是DMA地址）。中间过程可能需要pin住或者unpin用户态内存，常见处理都是用
`get_user_pages和put_page`等函数。

那现在要旁路CPU，而GPU中的代码执行的都是GPU的地址，这样就要求和其它设备和GPU是一个地址。
前文已经说过，GPU的内存是通过PCIe BAR空间呈现给CPU的，同理，在GPUDirect技术下，这个内存也要让其它
设备看看到。

## GPUDirect with RDMA

### 常规RDMA编程

再解释GPUDirect RDMA之前，先回顾下普通的RDMA程序是怎么写的。一般使用libibverbs这个库提供的API使用RDMA技术，基本分为以下几步：

	* Create an Infiniband context (struct ibv_context* ibv_open_device())
	* Create a protection domain (struct ibv_pd* ibv_alloc_pd())
	* Create a completion queue (struct ibv_cq* ibv_create_cq())
	* Create a queue pair (struct ibv_qp* ibv_create_qp())
	* Exchange identifier information to establish connection
	* Change the queue pair state (ibv_modify_qp()): change the state of the queue pair from RESET to INIT, RTR (Ready to Receive), and finally RTS (Ready to Send) 5
	* Register a memory region (ibv_reg_mr())
	* Exchange memory region information to handle operations
	* Perform data communication

其中最关键一步是register a memory region，用的时候通常是下面这么用，用户需要指定buffer的地址和大小：

		struct ibv_mr* registerMemoryRegion(struct ibv_pd* pd, void* buffer, size_t size) {
		  return ibv_reg_mr(pd, buffer, size, IBV_ACCESS_LOCAL_WRITE | IBV_ACCESS_REMOTE_READ | IBV_ACCESS_REMOTE_WRITE);
		}

### GPUDirect RDMA编程

GPUDirect RDMA的发展也分为几个阶段，在初期只是offload数据面，控制面还是在CPU，
后面又尽可能把控制面也给GPU，但是还是有小部分处理必须在CPU。

![gpudirect](/images/gpudirect.png)


#### GPUDirect RDMA

先看看Nvidia GPUDirect RDMA技术解决的问题：怎么让一个PCIe RP下的RDMA网卡可以直接访问GPU的内存，而避免把数据拷贝多次？

![nvidia_gpudirect](/images/nvidia_gpudirect.png)

由于GPU内存被RDMA网卡可见，所以写代码的时候，可以少申请一次malloc和memcpy。

![nvidia_gpudirect2](/images/nvidia_gpudirect2.png)

上面`ibv_reg_mr`注册内存的时候，居然允许注册GPU(peer to peer设备)的内存。
为了做到这一步，nvidia做了几下几步：

* 引入`io_peer_mem`这个ko模块，先把peer设备的内存暴露出来，并把peer设备作为client通过`ib_register_peer_memory_clent`注册到RDMA子系统中
* `io_peer_mem`主要实现三个回调函数:
	* acquire: 判断一个虚拟地址是否属于该peer设备
	* get_pages:获取这个memory region的物理地址
	* dma_map：获取这个memory region的iova地址

具体流程如下：

![nvidia_gpudirect3](/images/nvidia_gpudirect3.png)

到这里，已经可以让RDMA网卡访问GPU的内存了，为了让GPU和网卡并行起来，CPU仍然扮演了厚重的调度角色，而且GPU空转时间比较长。

![nvidia_gpudirect4](/images/nvidia_gpudirect4.png)

有人想把控制面也offload一部分，于是乎GPUDirect Async概念被提了出来。

![nvidia_gpudirect5](/images/nvidia_gpudirect5.png)

具体的用户态代码参考[libgdsync](https://github.com/gpudirect/libgdsync/tree/master/src)

#### GPUDirect Async

整体的逻辑如下:

![nvidia_gpudirect5](/images/nvidia_gpudirect5.png)

依赖的软件如下：

![nvidia_gpudirect6](/images/nvidia_gpudirect6.png)

整体软件栈：

![nvidia_gpudirect7](/images/nvidia_gpudirect7.png)

#### Nvidia NCCL

后来Nvidia基于上面互联通信这些技术，又提出了[NCCL概念](https://github.com/NVIDIA/nccl)。

![nvidia_nccl](/images/nvidia_ncll.png)

## GPUDirect with NVME

## Nvidia Magnum IO

有了上面这些网络加速、IO加速技术之后，Nvidia更进一步提出了Magnum IO技术，把这些都打包在了一起。

![nvidia_magnumio](/images/nvidia_magnumio.png)

![nvidia_magnumio2](/images/nvidia_magnumio2.png)

## 参考

* [Developing a Linux Kernel Module using GPUDirect RDMA](https://docs.nvidia.com/cuda/gpudirect-rdma/)
* [ParallelComputingSpring2015](https://wrfranklin.org/wiki/ParallelComputingSpring2015/cuda/nvidia/doc/pdf/)
* [GPU Direct IO with HDF5](https://hdfgroup.org/wp-content/uploads/2020/10/GPU_Direct_IO_with_HDF5-_John_Ravi.pdf)
* [gdrcopy](https://github.com/NVIDIA/gdrcopy)
* [GPUDirect RDMA and Green Multi-GPU Architectures](https://on-demand.gputechconf.com/gtc/2013/presentations/S3266-GPUDirect-RDMA-Green-Multi-GPU-Architectures.pdf)
* [Asynchronous Peer-to-Peer Device Communication](https://network.nvidia.com/pdf/prod_software/Mellanox_PeerDirect_Asynch_peer-to-peer_device_communication.pdf)
* [Advancing Communication Technologies and System Architecture to Maxime Performance and Scalability of GPU Accelerated Systems](https://mug.mvapich.cse.ohio-state.edu/static/media/mug/presentations/17/nvidia-mug-17.pdf)
* [Introduction to Programming Infiniband RDMA](https://insujang.github.io/2020-02-09/introduction-to-programming-infiniband/)
* [OFVWG:GPUDirect and PeerDirect](https://downloads.openfabrics.org/ofv/ofv_presentation_GPU.pdf)
* [RDMA over ML/DL and Big Data Frameworks](https://www.sc-asia.org/2018/wp-content/uploads/2018/03/1_1500_Ido_Shamay.pdf)
* [Accelerating IO in the Modern Data Center: Magnum IO Architecture](https://developer.nvidia.com/blog/accelerating-io-in-the-modern-data-center-magnum-io-architecture/)
