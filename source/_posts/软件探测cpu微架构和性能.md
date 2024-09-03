---
title: 软件探测cpu微架构和性能
author: Joy Xu
date: 2024-08-31 01:08:46
tags: [linux kernel, cpu microarchitecture, benchmark]
---

# 引子

几年前看到Dougall Johnson写的M1 Apple芯片微架构的材料，就一直想模仿，但是一直没有挤出时间来。

# 范围和背景

本文讲的芯片微架构主要是指流水线(pipeline)，现在芯片pipeline一般会有几个stage，通常包括fetch, decode, execute和commit，如下图所示：

![典型乱序CPU微架构](/images/uarch_ooo.png)

整个软件执行起来似乎如绿色箭头一样是顺序执行的，但其实decode和commit之间一般是乱序的。
decode后，会把指令顺序的放到reorder buffer中，告诉执行单元尽快干活，干完之后通知reorder buffer这个活干完了。之后由commit单元按照入对顺序把执行完的指令捞出来出队。

指令在执行过程中，会拆分成更小的单元micro-ops/uops/mops，能拆分成多少个也叫width，或者说解码宽度。
下图是ARM Neoverse V1的微架构，可以观察到每个单元的width，有多少个EU(执行单元)

![ARM Neoverse V1微架构](/images/uarch_neoverse_v1_block_diagram.svg)

再来一个苹果M1的对比:

![Apple M1微架构](/images/uarch_m1_firestorm.png)

本文软件探测的范围实际上就是想覆盖上图中所有的单元，包括各个Cache，ROB大小，EUs种类个数，LSU个数。
所有代码可以参考 [我的工具箱](https://github.com/joyxu/joyxu-linux-toolbox)。

# 参考

* [Performance Speed Limits](https://travisdowns.github.io/blog/2019/06/11/speed-limits.html)
* [uarch bench wiki](https://github.com/travisdowns/uarch-bench/wiki)
* [The Microarchitecture Behind Meltdown](https://blog.stuffedcow.net/2018/05/meltdown-microarchitecture/)
* [x86, x64 Instruction Latency, Memory Latency and CPUID dumps](http://users.atw.hu/instlatx64/)
* [realworldtech and its forum](https://www.realworldtech.com/)
* [The microarchitecture of Intel, AMD and VIA CPUs: An optimization guide for assembly programmers and compiler makers](https://agner.org/optimize/)
* [Intel Core Microarchitecture Pipeline](https://www.cnblogs.com/TaigaCon/p/7678394.html)
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
