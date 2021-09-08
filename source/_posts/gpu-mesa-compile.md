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

## GLSL转换成GLSL IR

GLSL的shader的处理入口在`_mesa_glsl_compile_shader`中，shader的源码首先经过词法分析`src/compiler/glsl_lexer.ll`和
语法分析`glsl_parser.yy`的通用Lex/Yacc处理，输出抽象语法树（AST），在经过`ast_to_hir.cpp`处理，转成GLSL IR。

## GLSL IR处理

GLSL IR在`ir.h`中定义了几个重要的类，一般分成指令类，条件控制类和函数类。

* `exec_node`，基础类，作为最小的执行单元，包含前后指针，分别指向之前和之后的指令。
* `ir_insturction`，所有指令的基础类
* `ir_rvalue`，右侧赋值类，是表达式的几率
* `ir_expression`，表达式
* `ir_texture`，纹理
* `ir_swizzle`，向量或者矩阵变换类
* `ir_dereference`，用于访问存在在变量、数组和数据结构中的值
* `ir_constant`，常所有基本类型的常量
* `ir_variable`，变量
* `ir_loop`，循环
* `ir_if`，条件控制

GLSL的IR调试一般通过`MESA_GLSL=dump`，比如 `MESA_GLSL=dump glmark2`。

## GLSL IR lower处理

lower处理都在`src/glsl/lower_*.cpp`文件中，主要作用是重写部分生成的IR。这个过程主要的动作就是遍历IR树，对于所有
叶子节点一般都有`visit`函数，非叶子节点都有`vist_enter`和`visti_leave`函数。

以`lower_instructions.cpp`为例，一般的优化过程都是用效率高的IR代替生成的IR。

## GLSL IR转换成NIR

这部分的处理在`st_link_nir`和`glsl_to_nir`中处理。之后通过`st_nir_opts`做lower的优化。


## AMD指令转换流程

NIR -> LLVM -> AMD GPU IR

AMD直接修改了LLVM源码，具体参考[LLVM AMD GPU改动](https://github.com/llvm/llvm-project/tree/main/llvm/lib/Target/AMDGPU)


## ARM Midgard/Bifrost转换流程

NIR -> BIR

MESA编译过程中会生成`bi_builder.h`，其中会定义从NIR到BIR指令的映射关系。代码流程如下：

		bifrost_compile_shader_nir
		  emit_cf_list
		    emit_block
		      bi_emit_instr
		        bi_emit_alu
			  bi_fma_to

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
