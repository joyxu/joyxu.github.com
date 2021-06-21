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

上图中TGSI基本已经不用，除了在virgl里面还用之外，基本都已经切换到LLVM或者厂家的自研编译器中。

## 为什么不直接编译成native ISA呢？

NIR的处理积累了大量经验，包含SSA处理，控制流处理。

## 编译优化在后端还是NIR？

答案是NIR。
2019年XDC上intel专家Jason Ekstrand特意强调intel花了4年时间把backend的优化又搬到NIR中来。

## 调试方法

* 参考[mesa 环境变量](https://docs.mesa3d.org/envvars.html), 比如加上`NIR_PRINT=1` 可以导出NIR到SSA的转换过程；
加上"AMD_DEBUG=vs"，可以导出NIR到SSA，以及LLVM优化的过程。

## 编译性能优化

* 最直观，看帧率和效果
* 执行shader-db看看前后优化结果
* 通过perf抓shader-db编译热点函数，修改对比。

## 编译优化趋势

2019年ACO(AMD compiler)被运营Steam的公司Valve(CS开发公司)提出来后，现在已经合入到MESA主线中，当前只支持AMD GPU。
Intel也提出IBC(intel backend-compiler)的概念。
优化的维度主要从时间和帧率考虑，使用的测试脚本可以看参考连接。


## 参考

* [A TRIP DOWN THE GPU COMPILER PIPELINE](https://gpuopen.com/wp-content/uploads/slides/GPUOpen_Let%E2%80%99sBuild2020_A%20Trip%20Down%20the%20GPU%20Compiler%20Pipeline.pdf)
* [Embedded Graphics Drivers in Mesa](https://elinux.org/images/1/1f/Embedded-drivers-mesa.pdf)
* [How to not write a back-end compiler](https://xdc2019.x.org/event/5/contributions/325/attachments/416/666/How_to_not_write_a_back-end_compiler.pdf)
* [NIR: A new compiler IR for Mesa](http://www.jlekstrand.net/jason/projects/mesa/nir-notes/)
* [A year of ACO](https://xdc2020.x.org/event/9/contributions/612/attachments/713/1313/xdc2020-a-year-of-aco.pdf)
* [Optimizing shader assembly instruction on Mesa using shader-db](https://blogs.igalia.com/apinheiro/2015/09/optimizing-shader-assembly-instruction-on-mesa-using-shader-db/)
* [Optimizing shader assembly instruction on Mesa using shader-db (II)](https://blogs.igalia.com/apinheiro/2015/09/optimizing-shader-assembly-instruction-on-mesa-using-shader-db-ii/)
* [火焰图：全局视野的Linux性能剖析](https://segmentfault.com/a/1190000023103508)
* [An introduction to Mesa’s GLSL compiler (I)](https://blogs.igalia.com/itoral/2015/03/03/an-introduction-to-mesas-glsl-compiler-i/)
* [The Valve-funded shader compiler 'ACO' is being queued up for inclusion in Mesa directly (updated: merged)](https://www.gamingonlinux.com/2019/09/the-valve-funded-shader-compiler-aco-is-being-queued-up-for-inclusion-in-mesa-directly)
* [ACO: A New Compiler Backend for RADV](https://lists.freedesktop.org/archives/mesa-dev/2019-July/221006.html)
* [Implementing Optimizations in NIR](https://xdc2019.x.org/event/5/contributions/323/attachments/432/685/IanRomanick-XDC2019-Implementing-Optimizations-in-NIR.pdf)
* [Lessons_from_Control_Flow_in_AMDGPU](https://llvm.org/devmtg/2020-09/slides/Hahnle-Evolving_convergent_Lessons_from_Control_Flow_in_AMDGPU.pdf)
* [Optimizing i965 for the Future](https://xdc2018.x.org/slides/optimizing-i965-for-the-future.pdf)
* [SHADERS IN RADEONSI DYNAMIC LINKING AND NIR IN RADEONSI DYNAMIC LINKING AND NIR](https://documents.pub/document/shaders-in-radeonsi-dynamic-linking-and-nir-in-radeonsi-dynamic-linking-and-nir.html)
* [TGSI for Mesa](https://winddoing.github.io/post/58638.html)
* [NIR introduction](https://people.freedesktop.org/~cwabbott0/nir-docs/intro.html)
* [GLSL compiler: Where we've been and where we're going (2015 Edition)](https://www.x.org/wiki/Events/XDC2015/Program/turner_glsl_compiler.pdf)
* [NIR on the Mesa i965 backend End](https://archive.fosdem.org/2016/schedule/event/i965_nir/attachments/slides/1113/export/events/attachments/i965_nir/slides/1113/nir_vec4_i965_fosdem_2016_rc1.pdf)
