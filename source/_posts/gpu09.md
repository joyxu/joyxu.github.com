---
title: Linux GPU系列-GPU驱动到底做什么2
author: Joy Xu
date: 2021-05-21 08:24:13
tags: [linux, gpu]
---

## GPU 内存分配和拷贝

注意到上篇中glGenbuffers，glBufferData过程中，涉及到gpu内存的分配和拷贝。

通常GPU都有自己的GDDR，要分配这部分内存的话，通常做法是内核态设备驱动实现mmap接口，
并把这块内存地址pin住，防止cpu去改动这块地址的映射关系。


## panfrost内存分配

基于mali panfrost来分析的话，用户态的代码在mesa的`src/panfrost/lib/pan_bo.c`中。

		panfrost_bo_create
		  panfrost_bo_alloc
		    drmIoctl  PANFROST_CREATE_BO

之后陷入kernel之后的流程：

		panfrost_ioctl_create_bo
		  drm_gem_shmem_create

之后再用户态就可以用普通的memcpy来拷贝了。


为了减少和kernel态的交互，mesa panfrost代码中也实现了一个内存pool，详细代码在
`src/panfrost/lib/pan_pool.c`中，使用最多的函数是panfrost_pool_alloc_aligned。

## 参考

* [linux DRM GEM 笔记](https://www.cnblogs.com/yaongtime/p/14418357.html)
* [年子GPU闲谈](https://jizhuoran.gitbook.io/mali-gpu/）

