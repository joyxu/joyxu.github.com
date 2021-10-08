---
title: Linux GPU系列-现在做GPU是不是太晚了？
author: Joy Xu
date: 2021-10-08 20:24:13
tags: [linux, gpu]
---

今天是2021年10月8号，从1999年Nvidia退出的第一个的GPU GeForce 256到现在已经差不多21年了，
现在开始做GPU是不是太晚了？

我觉得时机刚刚好，为什么这么说呢？

回答这个问题前，先简单回顾下GPU历史，再说下当前的形势，最后说下为什么我的回答是这样。

## GPU历史

Nvidia抓住了两次大的机遇，从而成为GPU的巨头。
第一次是1999年推出的第一个业界公认的GeForce 256，和微软推出的D3D API紧密合作，干掉3Dfx，从而垄断了大部分渲染市场，和ATI（AMD）平起平坐。
第二次是2007年基于斯坦福大学的BrookGPU，重新定义支持异构计算的GPU，结合自家的CUDA软件解决方案，吃掉了大部分计算市场，这次ATI(AMD)起了个大早却赶了个晚集
（ATI比Nvidia更早推出CTM异构计算框架）。

## 当前形势

### GPU硬件

随着人们对美好生活的向往：），以及摩尔定律（算力的突飞猛进），过去需要大量算力的Ray tracing渲染技术重新被
端了上来，2018年9月Nvidia推出第一代支持Ray Tracing的Turing架构GPU，到2020年的Ampere也就是第二代而已。而AMD
于2020年11月才推出第一代支持Ray Tracing架构的RDNA2。

### 标准

2018年10月，微软推出支持Ray Tracing（DirectX Ray Tracing）的D3D12。2020年5月才发布DXR Tier 1.1更新版本。
2020年12月,GPU标准组织Khronos才推出正式地支持Ray Tracing的Vulkan 1.2版本SDK。
OpenGL目前还看不到任何计划增加Ray Tracing。

### GPU软件

当前支持Ray Tracing的软件方案基本都是各家的，并没有形成统一的框架，比如Nvidia的GeForce RTX、OPTIX，AMD的RadeonRays，
intel的OSPRay、Embree技术，基本还是属于初期，百家争鸣的状态。

## 结论

GPU Ray Tracing的渲染时代才刚刚到来，对于新加入的GPU厂商来说，这是当下最好的时机，不会太早，也不晚，刚刚好。
