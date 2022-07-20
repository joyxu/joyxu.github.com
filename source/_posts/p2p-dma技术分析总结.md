---
title: p2p dma技术分析总结
author: Joy Xu
date: 2022-07-19 07:04:08
tags: [linux, p2pdma, dma-buf, hmm]
---
 
Google检索p2p dma能发现好多技术名词，什么p2pdma, dma-buf, GPUDirect, NVMEoF-P2P, SPIN, XDMA, Donard等等，
自然而然就陷入了疑惑，为什么会有这么多解决类似问题的方案呢？它们之间又是什么关系呢?又有什么区别呢？哪种场景下该用哪个技术呢？差异在哪里呢？

本文在AMD GPU的maintainer Alex Deucher总结材料上，补充些信息，做个总结。

## Out of tree方案(暂未何如到linux kernel主线)

### MLX的PeerDirect方案

之前已经介绍过这个技术，就不重复了，[官网地址](https://docs.nvidia.com/networking/pages/viewpage.action?pageId=58753175)。
这个方案依赖MLX定制的infiniband内核ko，相关的patch [Peer-direct memory](https://www.spinics.net/lists/linux-rdma/msg33294.html)并没有被主线接受,
开发过程可以参考这个[链接](https://review.gerrithub.io/plugins/gitiles/Artemy-Mellanox/io_peer_mem/)。

用户态的测试用例，可以参考修改后的[ib perftest](https://github.com/lsgunth/perftest/tree/mmap)。

参与这个方案开发的Stephen Bates(Donard系统的作者)后来成为了存储计算芯片公司Eideticom的CTO。

### Nvidia的GPUDirect方案

这个方案其实和MLX很有渊源，因为GPU的一个主要使用场景就是通过RDMA技术去访问远端的数据。nv_peer_memory的代码和MLX方案的io_peer_mem基本一致。

有兴趣的可以看看参考方案的[用例代码](https://github.com/Mellanox/gpu_direct_rdma_access)。
它依赖MLX定制的infiniband内核ko，[相关的patch](https://patchwork.kernel.org/project/linux-rdma/list/?series=&submitter=&state=*&q=IB%2Fcore%3A+Introduce+peer+client+interface&archive=both&delegate=)并没有被主线接受。
对端设备侧的驱动也需要修改，比如上例中使用了Nvidia的GPU，所以还依赖一个[nv_peer_memory的ko](https://github.com/Mellanox/nv_peer_memory),
它把GPU的内存通过`ib_register_peer_memory_client`函数注册到rdma的环境中。
另外在再做具体地址映射时，又依赖[nvidia-peermem.ko](https://github.com/NVIDIA/open-gpu-kernel-modules)。

### AMD的kfd_peerdirect方案

这个方案实际上是基于MLX的方案的，代码很相似，详细可以看看[源码](https://github.com/RadeonOpenCompute/ROCK-Kernel-Driver/blob/master/drivers/gpu/drm/amd/amdkfd/kfd_peerdirect.c)。


## In tree方案（已经何如到linux kernel主线）


## 参考

* [Seamless Operating System Integration of Peer-to-Peer DMA Between SSDs and GPUs](https://usenix.org/sites/default/files/conference/protected-files/atc17_slides_bergman.pdf)
* [Why is Peer to Peer DMA so hard on Linux?](https://lpc.events/event/9/contributions/617/attachments/705/1303/xdc2020_p2p_dma_v4_20200915_clean.pdf)
* [Coprocessor memory definition](https://openamp.github.io/docs/mca/coprocessor-memory-definition-v6.pdf)
* [Microsemi PCIE Switch+RDMA](https://www.microsemi.com/document-portal/doc_download/136008-microsemi-pcie-switch-rdma)
* [浅析nv_peer_memory的实现](https://zhuanlan.zhihu.com/p/439280640)
* [Building Mellanox OFED from source code](https://insujang.github.io/2020-01-25/building-mellanox-ofed-from-source/)
