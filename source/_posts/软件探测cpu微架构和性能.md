---
title: 软件探测cpu微架构和性能
author: Joy Xu
date: 2024-08-31 01:08:46
tags: [linux kernel, cpu microarchitecture, benchmark]
---

# 引子

几年前看到Dougall Johnson写的M1 Apple芯片微架构的材料，就一直想模仿，但是一直没有挤出时间来。

![典型乱序CPU微架构](/images/uarch_ooo.png)


![AP one CPU微架构](/images/uarch_apone.png)


# 参考

* [Performance Speed Limits](https://travisdowns.github.io/blog/2019/06/11/speed-limits.html)
* [uarch bench wiki](https://github.com/travisdowns/uarch-bench/wiki)
* [x86, x64 Instruction Latency, Memory Latency and CPUID dumps](http://users.atw.hu/instlatx64/)
* [realworldtech and its forum](https://www.realworldtech.com/)
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

