---
title: linux kernel调试
author: Joy Xu
date: 2024-01-30 03:38:00
tags: [kernel, debug]
---

## 引言

这个文章主要总结下，除了加打印，还有什么偷懒的办法来调试内核。
务必多读读Brendan Gregg的博客，可以说他是这个领域的第一人，另外也请多读阿里云杨勇的博客，他系统整理了性能观测的流程。

## 静态调试

是指不修改应用的源码，只是观察系统的运行状态，来了解当前的问题。

### 系统负载

通常有问题的时候，系统负载都是高的，一般可以通过top、w、uptime或者`cat /proc/loadavg`来了解系统的负载和负载趋势。
这个负载表示的是task数/core数，具体可以参考代码:[loadaverage.c](https://elixir.bootlin.com/linux/v6.7/source/kernel/sched/loadavg.c),
task数包括D状态(uninterruptible)和R状态(running)的task，所以这个负载已经不是单纯cpu的负载了，而是系统的负载,
也可以通过man命令 `man proc | sed -n '/loadavg/,/^$/ p'` 来了解具体含义。

       /proc/loadavg
       The first three fields in this file are load average figures giving the number of jobs in the run queue (state R) or waiting for disk I/O (state D) averaged over 1, 5, and 15 minutes.  They are the same as the load
       average  numbers  given  by uptime(1) and other programs.  The fourth field consists of two numbers separated by a slash (/).  The first of these is the number of currently runnable kernel scheduling entities (pro‐
       cesses, threads).  The value after the slash is the number of kernel scheduling entities that currently exist on the system.  The fifth field is the PID of the process that was most recently created on the system.

![linux debug load](/images/linux-debug-load.png)

如果1分钟的负载比5或者15分钟高，说明系统负载在增加，否则在减少。如果数值大于1，或许系统有问题。
既然R和D都给系统负载做了贡献，分析系统高负载的问题，首先要清楚是R带来的，还是D带来的。
整个debug流程大致如下：

![linux debug load](/images/linux-debug-flow.png)

#### R状态和D状态task变化趋势

可以通过以下ps命令观察R和D状态的变化

	ps -eo s,user,cmd | grep “^[RD]” | sort | uniq -c | sort -nbr | head -20

也可以通过我工具箱里的psn观察

![linux debug load](/images/linux-debug-load2.png)

#### top


## 动态跟踪调试(tracing)

是指通过动态跟踪工具，动态掌握执行的状态，来理解当前的问题。在linux上，tracing的全景图如下:

![linux tracing arch](/images/linux-tracing-big-picture.png)

个人推荐的是bpftrace和systemtap。

### systemtap

systemtap的历史比较悠久了，

### bpftrace

bpftrace是近几年火起来的工具，也有很多脚本可以直接使用了。

## 参考

* [内核主线讨论为什么要D状态](https://lore.kernel.org/lkml/Pine.LNX.4.33.0208011315220.12103-100000@penguin.transmeta.com/)
* [Brendan Gregg's Blog](https://www.brendangregg.com/blog/index.html)
* [Linux Load Averages: Solving the Mystery](https://www.brendangregg.com/blog/2017-08-08/linux-load-averages.html)
* [Linux tracing systems & how they fit together](https://jvns.ca/blog/2017/07/05/linux-tracing-systems/)
* [Full-system dynamic tracing on Linux using eBPF and bpftrace](https://www.joyfulbikeshedding.com/blog/2019-01-31-full-system-dynamic-tracing-on-linux-using-ebpf-and-bpftrace.html)
* [手艺人舍bpftrace而取systemtap的代价和思考](https://blog.csdn.net/dog250/article/details/112387192)
* [第一次使用Linux内核的Tracepoint的体验](https://blog.csdn.net/dog250/article/details/112058189)
* [Linux High Loadavg Analysis](https://oliveryang.net/2017/12/linux-high-loadavg-analysis-1/)
* [cpu load中所说的不可中断状态到底是啥？](https://www.pulpcode.cn/2021/10/04/what-is-uninterrupte-in-cpu-load/)
* [容器技术——Cgroup](https://blog.csdn.net/General_zy/article/details/132939514)
* [Linux 中 D 状态的进程与平均负载](https://cloud.tencent.com/developer/article/2002941)
* [进程hang在何处](https://zzyongx.github.io/blogs/where-is-process-hang.html)
* [Linux Process Snapper](https://tanelpoder.com/psnapper/)
* [High System Load with Low CPU Utilization on Linux?](https://tanelpoder.com/posts/high-system-load-low-cpu-utilization-on-linux/)
* [运行状态的进程和线程](https://huataihuang.gitbooks.io/cloud-atlas/content/os/linux/process/management/process_in_run_queue.html)
