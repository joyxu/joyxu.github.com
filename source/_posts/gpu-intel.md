---
title: intel gpu分析
author: Joy Xu
date: 2021-05-28 06:33:55
tags: [linux, gpu]
---

## intel gpu 介绍

intel的gpu一般是集显，也有pcie形态的独显，比如最新的DG1系列。
本文主要介绍intel的集显。

## intel 集显工作方式

和之前介绍的gpu工作方式基本一致，用户态准备数据通过ioctl接口让GPU驱动
把数据让GPU可见，同时把数据提交给GPU硬件，GPU执行完后通知驱动或者用户态。
抽象的流程如下：

![intel gpu_workflow](/images/gpu_intel_execbuf_steps.svg)

## 源码

kernel的驱动代码已经支持了当前最新的GEN12，还不支持独立显卡DG1。


* [kernel源码](https://git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git/tree/drivers/gpu/drm/i915/gem/i915_gem_execbuffer.c)

* [用户态测试代码](https://gitlab.freedesktop.org/drm/igt-gpu-tools/-/blob/master/tests/i915/gem_exec_nop.c)

## 参考

* [i915 command submission via gem_exec_nop](http://bwidawsk.net/blog/2013/8/i915-command-submission-via-gem_exec_nop/)
* [i915/GEM Crashcourse: Overview](https://blog.ffwll.ch/2013/01/i915gem-crashcourse-overview.html)
* [Intel vulkan driver introduction](https://wenxiaoming.github.io/2020/06/28/Intel%20vulkan%20driver%20introduction/)
* [Intel GPU 内存管理](http://liujunming.top/2019/07/16/Intel-GPU-%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/)
* [FFmpeg 在 Intel GPU 上的硬件加速与优化](https://toutiao.io/posts/w82yn0/preview)
* [Global Graphics Translation Table](http://blog.sina.com.cn/s/blog_8df488680102vosk.html)a
* [GEM Overview](https://blog.ffwll.ch/2011/05/gem-overview.html)
* [显卡内存管理机制及驱动实现Intel gma500为例](http://happyseeker.github.io/kernel/2016/07/22/memory-management-from-graphic-card-view.html)
* [System address map initialization in x86/x64 architecture part 1: PCI-based systems](https://resources.infosecinstitute.com/topic/system-address-map-initialization-in-x86x64-architecture-part-1-pci-based-systems/)
* [Linux graphic stack](https://www.studiopixl.com/2017-05-13/linux-graphic-stack-an-overview)
* [GPU存储体系-Integrated GPU](https://zhuanlan.zhihu.com/p/36030749)
