title: 'ARM GIC Linux Kernel接口分析(一)#'
author: Joy Xu
date: 2018-10-18 15:27:27
tags: [Linxu Kernel, IRQ, 中断]
---

这篇文章是基于[Linux-Kernel-ARM64体系中断系统架构跟踪](http://joyxu.github.io/2017/03/07/Linux-Kernel-ARM64%E4%BD%93%E7%B3%BB%E4%B8%AD%E6%96%AD%E7%B3%BB%E7%BB%9F%E6%9E%B6%E6%9E%84%E8%B7%9F%E8%B8%AA/)的第一个扩展。
很多内容都出自于[GICv3 and GICv4 Software Overview](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dai0492b/index.html)。

# 硬件初始化
GIC首先是一个中断控制器，自然要实现`irq_chip`:`gic_eoimode1_chip`，
同时也需要创建`irq_domain`来分配`irq_desc`，就要实现`irq_domain_ops`:`gic_irq_domain_ops`。

# GIC 中断种类
GIC规范中定义了4类中断类型:SGI(0-15), PPI(16-31), SPI(32-1029)和LPI(8192-~)。
各种类型的中断分配在`gic_irq_domain.gic_irq_domain_alloc`中实现。
这一块只是从ACPI表格、DTS表格读取hwirq配置和virq的关系而已。

# 中断路由配置


