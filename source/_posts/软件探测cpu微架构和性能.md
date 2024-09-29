---
title: 软件探测cpu微架构和性能
author: Joy Xu
date: 2024-08-31 01:08:46
tags: [linux kernel, cpu microarchitecture, benchmark]
---

# 引子

几年前看到Dougall Johnson写的M1 Apple芯片微架构的材料，就一直想模仿，但是一直没有挤出时间来。

# 背景

本文讲的芯片微架构主要是指流水线(pipeline)，现在芯片pipeline一般会有几个stage，通常包括fetch, decode, execute和commit，如下图所示：

![典型乱序CPU微架构](/images/uarch_ooo.png)

整个软件执行起来似乎如绿色箭头一样是顺序执行的，但现代处理器为了提升指令执行并发度，其实decode和commit之间是乱序执行的(out of order)。
decode后，会把指令顺序的放到reorder buffer中，告诉执行单元尽快干活，干完之后通知reorder buffer这个活干完了。之后由commit单元按照入对顺序把执行完的指令捞出来出队。
如果再把front end, back end，预取等概念加进来的话，更类似于下面这个图。

![典型乱序CPU微架构2](/images/uarch_ooo2.png)

指令在执行过程中，会拆分成更小的单元macro-ops(mops decode单元)和micro-ops(uops dispatch单元)，能拆分成多少个也叫width，或者说解码宽度(pipeline width)。
下图是ARM Neoverse V1的微架构，可以观察到每个单元的width，有多少个EU(执行单元)

![ARM Neoverse V1微架构](/images/uarch_neoverse_v1_block_diagram.svg)

再来一个苹果M1的对比:

![Apple M1微架构](/images/uarch_m1_firestorm.png)

这里再放一个ARM N1到N2的微架构演进的列表：

![ARM Neoverse N2微架构](/images/uarch_arm_n2.png)

![ARM Neoverse N2微架构2](/images/uarch_arm_n22.png)

# 范围

本文软件探测的范围实际上就是想覆盖上图中所有的单元，包括各个Cache，ROB大小，EUs种类个数，LSU个数。
所有代码可以参考 [我的工具箱](https://github.com/joyxu/joyxu-linux-toolbox)。

# Cache探测

探测之前，先科普下ARM上Cache的组织方式，具体如下图:

![ARM Cache 组织方式](/images/uarch_arm_cache_layer.png)

Cache探测除了组织方式以外，还包括时延和带宽，主要通过pointer chase这种算法，大部分主流的内存测试套比如lmbench均基于该算法。
算法的基本思想是创建一个数组，每个数组元素中保存一个指针，指向下一个要访问的数组元素，设置不同的步长，来遍历这个数组，观察时延的变化。
一般L1 Cache都是个位数的Cycle，L2 Cache大概十几个Cycle，L3 Cache大概50以内个Cycle左右，DDR基本都是上百个Cycle。
于是呢一旦时延变化较大，则肯定发生cache miss，就可以判断cache的大小了。

Cache Coherence的机制和实现可以参考[卡内基梅隆大学的Snooping-Based Cache Coherence](https://www.cs.cmu.edu/afs/cs/academic/class/15418-s21/www/lectures/11_cachecoherence1.pdf)

## Cache影响

Cache对软件的影响，主要有cache miss，false sharing(多core同时访问同一个cache line)，cache thrashing(多core同时访问不同way的同一个index的cache line，也就是set冲突)。
更多有趣的cache影响软件行为，可以参考陈浩的这篇文章[与程序员相关的CPU缓存知识](https://coolshell.cn/articles/20793.html)

# 执行单元个数

主要看不同种类指令的吞吐率，可以参考uarch bench，也就是参考链接中的第一篇文章。

# ROB大小

主要参考Henry Wang的测试ROB大小的文章，也放在参考链接中了。
通过填充NOP指令在两次cache miss的load中，来观察执行时间的变化，由于nop指令执行时间非常短，一旦时间变化很大，那么可以判断第二次cache miss的指令在ROB队列以外了。
代码实现可以参考[microarchitecturometer](https://github.com/Veedrac/microarchitecturometer.git)的实现。

# 总结

最后放一张为了提升性能的总版图：

![opt space](/images/uarch_opt_space.png)

# 参考

* [Performance Speed Limits](https://travisdowns.github.io/blog/2019/06/11/speed-limits.html)
* [uarch bench wiki](https://github.com/travisdowns/uarch-bench/wiki)
* [The Microarchitecture Behind Meltdown](https://blog.stuffedcow.net/2018/05/meltdown-microarchitecture/)
* [x86, x64 Instruction Latency, Memory Latency and CPUID dumps](http://users.atw.hu/instlatx64/)
* [realworldtech and its forum](https://www.realworldtech.com/)
* [uops info](https://uops.info/background.html)
* [The microarchitecture of Intel, AMD and VIA CPUs: An optimization guide for assembly programmers and compiler makers](https://agner.org/optimize/)
* [Intel Core Microarchitecture Pipeline](https://www.cnblogs.com/TaigaCon/p/7678394.html)
* [CPU Introspection: Intel Load Port Snooping](https://gamozolabs.github.io/metrology/2019/12/30/load-port-monitor.html)
* [AmpereOne at Hot Chips 2024: Maximizing Density](https://chipsandcheese.com/2024/08/29/ampereone-at-hot-chips-2024-maximizing-density/)
* [computer microarchitecture之out-of-order execution](https://blog.csdn.net/wanjia19870902/article/details/108005731)
* [Apple Microarchitecture Research by Dougall Johnson](https://dougallj.github.io/applecpu/firestorm.html)
* [Measuring Reorder Buffer Capacity](https://blog.stuffedcow.net/2013/05/measuring-rob-capacity/)
* [Memory Performance in a Nutshell](https://www.intel.cn/content/www/cn/zh/developer/articles/technical/memory-performance-in-a-nutshell.html?wapkw=memory%20performance%20in%20a%20nutshell)
* [Finding Your Memory Access Performance Bottlenecks](https://www.intel.cn/content/www/cn/zh/developer/articles/technical/finding-your-memory-access-performance-bottlenecks.html?wapkw=memory%20performance%20in%20a%20nutshell)
* [computer architecture - microarchitectural channels](https://portrait.gitee.com/joey-fudan/cpplinks/blob/master/comparch.micro.channels.md#return-stack-buffer-rsb)
* [performance tools](https://portrait.gitee.com/joey-fudan/cpplinks/blob/master/performance.tools.md)
* [The Validation Buffer Microarchitecture for Multithreaded Processor](https://xueshu.baidu.com/usercenter/paper/show?paperid=cfa7f6ebfda9d6db3ae578b855c50498&site=xueshu_se)
* [Neoverse V1 - Microarchitectures - ARM](https://en.wikichip.org/wiki/arm_holdings/microarchitectures/neoverse_v1)
* [移动芯片排行](https://socpk.com/)
* [Arm's New Cortex-A78 and Cortex-X1](https://cloud.tencent.com/developer/article/2003298)
* [Apple M1](https://www.anandtech.com/show/16226/apple-silicon-m1-a14-deep-dive)
* [ARM processor](http://medium.com/vswe/arm-processor-80ac96be881a)
* [RPi 4 tuning: The code](https://sandsoftwaresound.net/rpi-4-tuning-the-code/)
* [《计算机体系结构：量化研究方法》 第1章 量化设计和分析的基础知识（一）](https://zhuanlan.zhihu.com/p/683781147)
* [Computer Architecture Research with RISC-V - SonicBOOM: The 3rd Generation Berkeley Out-of-Order Machine](https://carrv.github.io/2020/)
* [如何设计一个成功的指令集架构](https://martins3.github.io/cpu/arch-design.html)
* [Arm Mali GPU Training Series Ep 1.4 Hardware shader cores](https://www.bilibili.com/video/BV1yo4dexEeb/?spm_id_from=333.788)
* [A Metric-Guided Method for Discovering Impactful Features and Architectural Insights for Skylake-Based Processors](https://www.researchgate.net/publication/338028324_A_Metric-Guided_Method_for_Discovering_Impactful_Features_and_Architectural_Insights_for_Skylake-Based_Processors?_sg=Fd4reHj2JbXbVz1CZCmAIcT3PZLC2T9ttDv96ylqp01DpjSkstBb2LLfwBSlwPfFG7HfWGNq4Zc-jEI&_tp=eyJjb250ZXh0Ijp7ImZpcnN0UGFnZSI6Il9kaXJlY3QiLCJwYWdlIjoiX2RpcmVjdCJ9fQ)
* [Understanding CPU Microarchitecture to Increase Performance](https://www.infoq.com/presentations/microarchitecture-modern-cpu/)
* [Zen 5’s Leaked Slides](https://chipsandcheese.com/2023/10/08/zen-5s-leaked-slides/)
* [Lecture 7 (2-1-05) Superpipelining and Superscalar Architecture Overheads](https://www.cs.uni.edu/~fienup/cs240s05/lectures/lec7_2-1-05.htm)
* [DRILLING DOWN INTO THE XEON SKYLAKE ARCHITECTURE](https://www.nextplatform.com/2017/08/04/drilling-xeon-skylake-architecture/)
* [Customizing Processors](https://semiengineering.com/customizing-processors/)
* [A Journey Through the CPU Pipeline](http://gamedev.net/tutorials/programming/general-and-gameplay-programming/a-journey-through-the-cpu-pipeline-r3115/)
* [Memory Barriers: a Hardware View for Software Hackers](https://www.researchgate.net/publication/228824849_Memory_Barriers_a_Hardware_View_for_Software_Hackers)
* [Cache: a place for concealment and safekeeping](https://manybutfinite.com/post/intel-cpu-caches/)
* [17_ARMv8_高速缓存（二）ARM cache设计](https://github.com/carloscn/blog/issues/58)
* [从技术角度聊CPU分支预测器](https://zhuanlan.zhihu.com/p/715411484)
* [卡内基梅隆大学的Parallel Computer Architecture and Programming, Spring 2024](https://www.cs.cmu.edu/afs/cs/academic/class/15418-s24/www/schedule.html)
* [Cache snoop](https://www.am.ics.keio.ac.jp/comparc/snoop.pdf)
* [The microarchitecture of Intel, AMD and VIA CPUs: An optimization guide for assembly programmers and compiler makers](https://agner.org/optimize/)
* [Intel 海光 鲲鹏920 飞腾2500 CPU性能对比](https://www.cnblogs.com/88223100/p/Comparison-of-CPU-Performance-between-Intel_Haiguang_Kunpeng-920_and-Feiteng-2500.html)
