---
title: 'io_uring, ebpf and kTLS'
author: Joy Xu
date: 2023-07-18 06:35:4:"vs
tags: [io_uring, eBPF, kTLS]
---

## IO系统调用

一个简单的read的流程如下图，其中会调用`syscall read`，这就是Linux Kernel的系统调用。

![IO 类别](/images/io_read.png)

可以通过以下命令查看系统调用

	sudo apt-get install -y auditd 
	ausyscall aarch64 --dump

也可以查看Linux Kernel的源码[unistd.h](https://git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git/tree/include/uapi/asm-generic/unistd.h)

## IO类型

如果按照同步、异步、阻塞和非阻塞来分Linux Kernel的IO方式，可以把涉及到IO系统调用划分成下面几类：

![IO 类别](/images/io_types.png)

## 参考

* [Implementing a Kernel Module to Eliminating External Caching Effects](https://cross.ucsc.edu/news/blog/mbwuconstruction_012020.html)
* [nofscache](https://ljishen.github.io/nofscache/)
* [ARM64 syscall table](https://blog.xhyeax.com/2022/04/28/arm64-syscall-table/)
