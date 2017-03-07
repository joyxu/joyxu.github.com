---
title: Linux Kernel ARM64体系中断子系统跟踪
author: "Joy Xu"
date: 2017-03-07 09:44:44
tags: [Linxu Kernel, IRQ, 中断]
---

# 目的
这篇文章基于linux kernel 4.10介绍ARMv8体系架构下，中断子系统的整体设计，并会持续跟踪下去。

# 读者
本文假设读者已有linux kernel编程经验，或者至少已经读过kernel document中IRQ相关的文档。

# 总述
最早的8251单片机里面，中断初始化主要分三步：定义中断号，定义中断函数和定义中断向量表。
linux kernel也不外乎如此，只不过为了支持不同的体系结构，不同的硬件设备，做了复杂的抽象。
kernel通过`virq`和`hw irq`联系起来，通过`virq`和`irq_desc`把`hw irq`和中断处理函数联系起来，
通过`irq_chip`和`irq_domain`把中断控制器的层级关系建立起来。

在ARM64上，随着平台设备和PCI设备的MSI中断的引入，整个子系统越来越复杂，没有一个整体脉络
的认识，很难理解背后代码变动的含义。
　　
# ARM64中断子系统整体设计
深入之前，再强调下几个认知：
* linux kernel中通过一个全局的`irq_desc`数组来保存中断信息
* 一个hw irq对应一个irq或者virq，`hw irq`是可以在不同`irq_domain`重复的
* irq或者virq通常是`irq_desc`数组的index

基于4.10的代码，它们之前的关系如下：

![ARM64中断子系统类图](http://omeik3jj4.bkt.clouddn.com/irq-classes.png)

再不考虑多个中断控制器的情况下，物理关系如下：

![单个中断控制器物理连接图](http://omeik3jj4.bkt.clouddn.com/irq_single.png)

物理中断触发之后，处理逻辑如下：

![中断处理逻辑图](http://omeik3jj4.bkt.clouddn.com/irq_desc.jpg)

# 引入多个中断控制器（比如GPIO和多GIC）之后，处理逻辑如下：　

![多中断处理逻辑图](http://omeik3jj4.bkt.clouddn.com/irq_multi.png)
![多中断处理逻辑图2](http://omeik3jj4.bkt.clouddn.com/irq_stack.png)

# 引入MSI中断之后

![msi中断处理逻辑图](http://omeik3jj4.bkt.clouddn.com/irq_msi.png)


# 引入平台设备MSI中断之后

![平台msi中断处理逻辑图](http://omeik3jj4.bkt.clouddn.com/irq_msi_bridge.png)

# 现在的domain状态

![domain图](http://omeik3jj4.bkt.clouddn.com/irq_complex.png)

# SMMU处理的例子

![smmu图](http://omeik3jj4.bkt.clouddn.com/irq_example.png)
