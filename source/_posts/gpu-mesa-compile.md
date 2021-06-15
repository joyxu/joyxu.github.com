---
title: GPU MESA 编译分析
author: Joy Xu
date: 2021-06-15 11:09:15
tags: [linux, gpu]
---

## MESA3D编译架构

整个架构如下

![mesa gpu compile arch](/images/gpu_mesa_compile_arch.png)

如果是Vulkan，shader从SPIRV先编译成NIR，再编译成native。
如果是OpenGL，则从GLSL先编译成NIR，再编译成native。

## 为什么不直接编译成native ISA呢？

NIR的处理积累了大量经验，包含SSA处理，控制流处理。

## 编译优化在后端还是NIR？

答案是NIR。
2019年XDC上intel专家Jason Ekstrand特意强调intel花了4年时间把backend的优化又搬到NIR中来。

## 编译性能优化

* 最直观，看帧率和效果
* 执行shader-db看看前后优化结果
* 通过perf抓shader-db编译热点函数，修改对比。

## 编译优化趋势

2019年ACO提出来后，现在已经和入到MESA主线中，并且被AMD使用。

## 参考

* [Embedded Graphics Drivers in Mesa](https://elinux.org/images/1/1f/Embedded-drivers-mesa.pdf)
* [How to not write a back-end compiler](https://xdc2019.x.org/event/5/contributions/325/attachments/416/666/How_to_not_write_a_back-end_compiler.pdf)
* [NIR: A new compiler IR for Mesa](http://www.jlekstrand.net/jason/projects/mesa/nir-notes/)
* [A year of ACO](https://xdc2020.x.org/event/9/contributions/612/attachments/713/1313/xdc2020-a-year-of-aco.pdf)
* [Optimizing shader assembly instruction on Mesa using shader-db](https://blogs.igalia.com/apinheiro/2015/09/optimizing-shader-assembly-instruction-on-mesa-using-shader-db/)
* [Optimizing shader assembly instruction on Mesa using shader-db (II)](https://blogs.igalia.com/apinheiro/2015/09/optimizing-shader-assembly-instruction-on-mesa-using-shader-db-ii/)
* [火焰图：全局视野的Linux性能剖析](https://segmentfault.com/a/1190000023103508)
