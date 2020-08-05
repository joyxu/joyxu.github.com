
---
title: Linux Tracers杂谈
author: "Joy Xu"
date: 2020-08-05 19:44:44
tags: [Linxu Kernel]
---

准备开始做性能分析的时候，网上找了下linux性能分析工具，结果发现一堆，什么LTTng，
eBPF，Systemtap，kprobe，FTrace和Perf等等，对于平时只用过perf的我，感觉一脸懵。

到底新手应该先掌握哪个呢？个人认为先掌握perf trace和eBPF，有时间可以再看看其它的。

这些工具的分类，可以从下面这些维度划分：
1. 数据源
2. 数据收集机制
3. 前端工具

#数据源

##probe

程序执行的时候，通过动态修改程序执行的汇编代码来获取程序运行的踪迹。典型的有
kprobe和uprobe。

##tracepoint

事先在程序的源码中打桩，编译成二进制之后，通过过滤和开关或许程序运行的踪迹，
比如kernel tracepoints。

#数据收集机制

##ftrace文件系统

通过读写/sys/kernel/debug/tracing文件系统获取trace数据

##perf

通过perf_event_open系统调用从perf的ring buffer获取trace数据， 比如perf trace

##eBFP

通过eBFP把自己写的eBPF程序通过probe机制注入kernel之后，在该程序中往eBFP映射的
map，ftrace或者perf buffer中写数据。

##systemtap

类似eBPF，通过probe方式注入自己写的程序之后，该程序通过relayfs往用户态传数据。

#前端工具

##perf
最常用的就是perf，比如perf trace

##bcc
配合eBPF使用

#参考
* [Linux tracing systems & how they fit together](https://jvns.ca/blog/2017/07/05/linux-tracing-systems/)
* [Choosing a Linux Tracer 2015](http://www.brendangregg.com/blog/2015-07-08/choosing-a-linux-tracer.html)
