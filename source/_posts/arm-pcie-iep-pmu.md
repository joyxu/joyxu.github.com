---
title: ARM PCIe IEP设备DFX思考
author: Joy Xu
date: 2020-06-03 14:18
tags: [Linux Kernel]
---

本文记录下如何实现ARM PCIE IEP外设DFX的软件方案。

# 背景

接触的ARM芯片上已经集成了好几种iEP外设，既存的方案是通过debugfs文件系统来使用
这些DFX寄存器。
之前ARM上没有uncore的pmu，最近的linux kernel主线上已经支持了uncore pmu，于是有
兄弟想是不是可以把这些iEP设备注册到perf的框架上，通过perf来使用这些DFX寄存器。
一个明显的好处是可以通过perf这个大家比较熟悉的工具来统一用户perf接口。

## perf 简介

perf的使用可以参考下面几个链接：
* [在linux下做性能分析3: perf](https://zhuanlan.zhihu.com/p/22194920)
* [linux效能分析工具](http://wiki.csie.ncku.edu.tw/embedded/perf-tutorial)

perf的工作原理主要是通过perf recode采样记录发生的事件，再通过perf report展现结果。

perf能记录的事件分为三类(可以通过perf list查看)：
* hardware：由硬件产生的事件，比如cpu-cycles
* software：由软件产生的事件，比如context-switches
* tracepoint：由linux kernel的静态tracepoint所触发的事件


# PMU框架方案思考

接触的ARM芯片上，IEP外设还不支持perf recode针对函数级别进行采样，只能通过perf 
stat针对hardware事件进行时间采样。

本文暂不考虑是否合适注册到Linux Perf框架上，只考虑如何利用Linux Perf框架。
对应的驱动可以参考:
https://elixir.bootlin.com/linux/latest/source/drivers/perf/hisilicon/hisi_uncore_l3c_pmu.c

主要的步骤：
1. 实现PMU结构体，特别是start,stop和read三个回调函数，具体参考
https://elixir.bootlin.com/linux/latest/source/include/linux/perf_event.h#L269
2. 调用perf_pmu_register注册到perf框架
3. 申请中断和注册中断函数(非必须，hisi的驱动主要是担心硬件计数器溢出)
