---
title: ARM Virtualisation
author: Joy Xu
date: 2017-05-22 16:36:49
tags: [Virtualization, Linux Kernel]
---

#目的

* 基于虚拟化理论要求，解释ARM VHE扩展
* 基于虚拟化实现模型，解析KVM实现要点
* 记录后续虚拟化的发展方向

#虚拟化理论基础

根据[Popek and Goldberg virtualization requirements](https://en.wikipedia.org/wiki/Popek_and_Goldberg_virtualization_requirements)理论，
实现虚拟化的一个充分条件是针对ＶＭＭ提供一套特权指令集来控制虚拟化资源，同时该理论也提供了一个简单的技术实现方案:trap-and-emulate。


todo: trap-and-emulate 架构图片

#ARM VHE 扩展

##v8.1扩展
* 增加EL2，新增HYP模式，在该模式下处理VMM相关特权指令控制虚拟化资源，新增HYP模式中断向量表(HVBAR)，CPU通过HVC或者SMC指令陷入HVC模式，并通过ERET返回EL1

todo: Host HYP Guest 转换

* 新增IPA概念，增加MMU stage 2转换，通过SMMU stage2实现IPA到PA转换，所有TLB和Cache操作均带VMID，host上VMID为0。

todo: VA->IPA->PA转换图

* 新增VGIC概念来加速IO中断虚拟化

* 新增Hypervisor Syndrome Regiser，HYP模式下，CPU可以通过该寄存器获取进入HYP模式的原因。
* 不支持nested
* stage 2 页表类似LPAE

##v8.3扩展

* fdasfsa
* fdsa范德萨

#整体架构

* VM lecture

#技术选型

##当前主流ARM虚拟化技术框架

* KVM
	* storage: qemu storage stack overview
* Xen
* Docker

#后续技术深入方向

	* network: hwanju io virtualzation
	* gpu: hwanju io virtualization
	* memory: hwanju memory task aware 
	* cpu: hwanju task aware schedule
	* best for virtuazliation: IBM


#依赖

* UEFI必须使能HYP模式，并让Linux kernel从HYP模式启动
* 只支持4KB页表

#方案评估策略

#附录
* network: macvlan
* storage: device mapper
* virtio: 
