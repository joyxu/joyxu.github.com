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
实现虚拟化的一个充分条件是针对ＶＭＭ提供一套特权指令集，同时该理论也提供了一个简单的技术实现方案:trap-and-emulate。

#ARM VHE 8.1扩展

* 增加PL2，新增hyp模式，新增ＨＹＰ模式中断向量表，ＣＰＵ通过ＨＶＣ或者ＳＭＣ指令陷入该模式，并通过ＥＲＥＴ返回
* 增加MMU stage 2转换，所有TLB和Ｃａｃｈｅ操作均带ＶＭＩＤ，ＶＭＩＤ
* 
* 
