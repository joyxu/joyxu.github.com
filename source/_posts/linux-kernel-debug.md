---
title: linux kernel调试
author: Joy Xu
date: 2024-01-30 03:38:00
tags: [kernel, debug]
---

## 引言

这个文章主要总结下，除了加打印，还有什么偷懒的办法来调试内核。
务必多读读Brendan Gregg的博客，可以说他是这个领域的第一人。

## 静态调试

是指不动应用的源码，只是观察系统的运行状态，来了解当前的问题。

## 动态跟踪调试(tracing)

是指通过动态跟踪工具，动态掌握执行的状态，来理解当前的问题。在linux上，tracing的全景图如下:

![linux tracing arch](/images/linux-tracing-big-picture.png)

个人推荐的是bpftrace和systemtap。

### systemtap

systemtap的历史比较悠久了，

### bpftrace

bpftrace是近几年火起来的工具，也有很多脚本可以直接使用了。

## 参考

* [Brendan Gregg's Blog](https://www.brendangregg.com/blog/index.html)
* [Linux tracing systems & how they fit together](https://jvns.ca/blog/2017/07/05/linux-tracing-systems/)
* [Full-system dynamic tracing on Linux using eBPF and bpftrace](https://www.joyfulbikeshedding.com/blog/2019-01-31-full-system-dynamic-tracing-on-linux-using-ebpf-and-bpftrace.html)
* [手艺人舍bpftrace而取systemtap的代价和思考](https://blog.csdn.net/dog250/article/details/112387192)
* [第一次使用Linux内核的Tracepoint的体验](https://blog.csdn.net/dog250/article/details/112058189)
