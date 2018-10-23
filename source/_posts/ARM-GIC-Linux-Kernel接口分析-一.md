title: 'ARM GIC Linux Kernel接口分析(一)'
author: Joy Xu
date: 2018-10-18 15:27:27
tags: [Linxu Kernel, IRQ, 中断]
---

这篇文章是基于[Linux-Kernel-ARM64体系中断系统架构跟踪](http://joyxu.github.io/2017/03/07/Linux-Kernel-ARM64%E4%BD%93%E7%B3%BB%E4%B8%AD%E6%96%AD%E7%B3%BB%E7%BB%9F%E6%9E%B6%E6%9E%84%E8%B7%9F%E8%B8%AA/)的第一个扩展。
很多内容都出自于[GICv3 and GICv4 Software Overview](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dai0492b/index.html)。

# 中断控制器

中断控制器在系统中的位置：

![线中断](/images/dedicated-irq.PNG)

![消息中断](/images/msi-irq.PNG)

# GIC硬件配置

GIC硬件可以分为三部分：GICD，GICR和GICC。

![GICv3编程模型](/images/gicv3-model.PNG)

GIC规范中定义了4类中断类型:SGI(0-15), PPI(16-31), SPI(32-1029)和LPI(8192-~)。
其中GICD主要管理SPI类型的中断，包括使能/不使能、优先级、路由到哪个GICR、边沿触发还是水平触发和SPI中断状态等；
GICR主要管理SGI、PPI和LPI类型的中断，包括使能/不使能、优先级、边沿触发还是水平触发、中断状态等；
GICC主要管理中断处理控制、ack、deack、PE中断mask等。

如果脱离kernel，GIC的配置主要围绕GIC和PE来配置:

* GICD:

	路由使能：GICD_CTLR ARE
	分发使能：GICD_CTLR EnabledGrp0

* GICC：

	使能系统访问： ICC_SRE_ELn
	优先级: ICC_PMR_EL1, ICC_BPRn_EL1
	EOI模式： ICC_CTRL_EL1
	中断使能: ICC_IGRPEN0_EL1

* PE:

	中断向量设置: VBAR_EL1

# Kernel中GIC使能


## 初始化

GIC首先是一个中断控制器，自然要实现`irq_chip`:`gic_eoimode1_chip`，
同时也需要创建`irq_domain`来分配`irq_desc`，就要实现`irq_domain_ops`:`gic_irq_domain_ops`。

另外GIC硬件初始化的部分在probe的时候, 通过`gic_init_base`实现上面讲的硬件配置部分，具体在：

		gic_init_base->
			gic_dist_init: gicd 配置，默认配置所有中断都路由到boot core
			gic_cpu_init: gicc 配置


## 中断分配

各种类型的中断分配在`gic_irq_domain.gic_irq_domain_alloc`中实现。
这一块只是从ACPI表格、DTS表格读取hwirq配置和virq的关系而已。

		msix_capability_init->
			pci_msi_setup_msi_irqs->
				msi_domain_alloc_irqs->
					irq_domain_alloc_irqs_hierarchy->
						irq_domain_alloc_irqs_parenet->
							gic_irq_domain_alloc

		irq_create_fwspec_mapping->
			irq_domain_alloc_irqs->
				__irq_domain_alloc_irqs->
					irq_domain_alloc_irqs_hierarchy->
						gic_irq_domain_alloc


## 中断路由配置

		request_irq->
			request_threaded_irq->
				__setup_irq->
					irq_startup->
						irq_setup_affinity->
							msi_domain_set_affinity->
								irq_chip_set_affinity_parent->
									gic_set_affinity

		irq_set_affinity*->
			irq_do_set_affinity->
				msi_domain_set_affinity->
					irq_chip_set_affinity_parent->
						gic_set_affinity

## 中断响应

		el1_irq->
			gic_handle_irq->
				gic_read_iar
				handle_domain_irq->
					generic->handle_irq->
						handle->
							gic_eoi_irq
