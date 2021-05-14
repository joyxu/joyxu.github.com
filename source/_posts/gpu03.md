---
title: Linux GPU系列-03-ARM MALI GPU工作流程介绍
author: Joy Xu
date: 2021-05-12 01:30:11
tags: [linux, gpu]
---

先讲讲ARM MALI GPU的工作流程，为后续做下铺垫。

一般GPU的pipeline主要有：
* Vertex处理，做MPV（Model，View， Project trasform）和Screen mapping坐标变换，clip裁剪
* Rasterization处理，主要遍历三角形
* Fragment处理，差值计算处理顶点颜色，纹理贴图等

如下图所示：
![GPU pipeline](/images/gpu_pipeline.png)

![GPU pipeline2](/images/gpu_pipeline2.png)

## ARM MALI GPU工作流程

由于MALI是基于TBR(Tiled based rendering)所以多一个Tiling或者binning的步骤。

如下图

![ARM T880 Tile](/images/t880_tile.png)

Tile和Vertext、Fragement间的交互如下：

![ARM T880 pipeline](/images/t880_pipeline.png)

## ARM MALI Midgard硬件单元框架

![Midgard的逻辑框图](/images/mali_midgard_blocks.png)

![Midgard软件接口](/images/mali_midgard_jobs_interface.png)

驱动把job提交给job manager，再由job manager分发给具体的硬件单元执行;job存在之前说的ring buffer中。
job有几种类型：

![Midgard软件接口](/images/mali_midgard_jobs.png)

job manager根据job类型分发：

![Midgard软件接口](/images/mali_midgard_job_dispatch.png)

## 结合OpenGL api观察整个流程

![Midgard软件接口](/images/mali_midgard_opengl.png)

OpenGL API主要组织好job所需的数据，为后续的shader core计算做好准备。
![Midgard软件接口](/images/mali_midgard_opengl_driver.png)

## 再补充些概念

* binning pass:对于TBR来说，就是拆成primitive的过程
* rendering pass：对每个primitive进行渲染着色的过程，一个rendering pass可以有多个sub pass，更精准的描述可以参考下面stackflow的答复

	The term "render pass" is more nebulous. The most common meaning refers to multipass rendering techniques.
	In multipass techniques, you render the same "object" multiple times, with each rendering of the object doing
	a separate computation that gets accumulated into the final value. Each rendering of the object with a particular
	set of state is called a "pass" or "render pass".

	Note that a render pass is not necessarily a draw call. Objects could require multiple draw calls to render.
	While this is typically slower than making a single draw call, it may be necessary for various reasons.

* draw call：对应到Open GL里面gl*Draw*这样的API，这些API的调用触发vertex被渲染，但是同时针对一个vertex调用多次API，仍然是一个draw call。

![MALI PASS](/images/mali_pass.png)

![QCOM Adreno PASS](/images/adreno_pass.png)

# 参考

[Shader Learing Render Pipeline篇](https://hushengstudent.blog.csdn.net/article/details/59122183)
[T880 Mobile GPU](https://pdfs.semanticscholar.org/6eea/4efe677304b6c77008e15d34ac39f1164e9e.pdf)
[ARM Midgard Architecture](https://fileadmin.cs.lth.se/cs/Education/EDAN35/guestLectures/ARM-Mali.pdf)
[The Bifrost GPU architecture and the ARM Mali-G71 GPU](https://old.hotchips.org/wp-content/uploads/hc_archives/hc28/HC28.22-Monday-Epub/HC28.22.10-GPU-HPC-Epub/HC28.22.110-Bifrost-JemDavies-ARM-v04-9.pdf)
[Optimizing Roblox: Vulkan Best Practices for Mobile Developers](https://zeux.io/data/gdc2020_arm.pdf)
[Adreno GPU Architecture](https://blog.csdn.net/Q1302182594/article/details/82767719)
[What exactly is a GPU binning pass](https://stackoverflow.com/questions/34196144/what-exactly-is-a-gpu-binning-pass)
[What is a renderpass?](https://stackoverflow.com/questions/34382340/what-is-a-renderpass)
[渲染流程](https://www.cnblogs.com/llstart-new0201/p/11949743.html)
[Unity - DrawCall, Batch, SetPassCall区别](https://blog.csdn.net/linjf520/article/details/113702206?spm=1001.2014.3001.5501)
[Render Hell 2.0](http://simonschreibt.de/gat/renderhell/)
