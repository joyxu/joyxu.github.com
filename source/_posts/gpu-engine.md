---
title: 游戏引擎介绍
author: Joy Xu
date: 2021-07-15 02:04:56
tags: [linux, gpu]
---

## 总纲

游戏引擎涉及的概念很多，总体如下：

![game engine](/images/game_engine_overall.png)

游戏引擎的历史大概如下，引擎的概念慢慢独立出来，不再为单一游戏服务。

![game engine history](/images/game_engine_hisitory.png)

当前比较热的游戏引擎注意是Unitiy和Unreal，刚好代表着商业和开源两大巨头。

![game engine list](/images/game_engine_list.png)

## 游戏制作流水线

随着游戏引擎的独立，整个产业链的分工也开始细化，数据和逻辑逐渐拆分开，专业的铲子工具也越来越多。
比如图片、模型，材质和sahder等。

游戏制作的流水线大体如下：

![game engine pipeline](/images/asset_timeline.jpg)

其中和程序员关联比较大的是模型这块的工具，常用的模型工具有收费的Autodesk和免费的blender，目前还是Autodesk占主导地位。
模型主要指静态模型，存储顶点数据，通常叫mesh，后缀一般是fbx，或者obj。

## 游戏引擎技术点

主要包括以下几个方面：

### 图片

由于GPU和CPU识别的图片格式不一致，也为了节约GPU的显存或者GPU的带宽，都会把CPU可见的png/jpg格式等保存为GPU认识的格式，比如`GL_COMPRESSED_RGBA`等定义到opengl规范中的格式。
这个功能也内置到Utility中了，这也就是为什么在Utility中导入图片会感觉很慢。

另外硬件厂商也在推各自的GPU识别的格式，比如ARM从ETC->ETC2->ASTC的演进。

### 静态模型（Mesh）

简单的渲染很容易把顶点数据写在逻辑中，数据分离后，模型顶点数据保存在mesh文件中。
模型导出mesh已经是前面说的专业工具做的事情了，只是blender、Autodesk等工具也支持编程实现。

### 材质

模型的渲染离不开材质，数据分离后，材质实际上可以认为是一个文本文件，记录Opengl参数，shader名字和shader需要的参数等。

### 多相机

变化的物体和静态的物体分离，避免无用的渲染(drawcall)。

## 参考

* [Game Engine](https://www.cs.kent.edu/~ruttan/GameEngines/lectures/compiling.pdf)
* [this is game](http://www.thisisgame.com.cn/book/makegameengineatnight/)
* [System Architectures](https://homepages.fhv.at/thjo/lecturenotes/sysarch/)
* [Virtual Reality for Architects - Sketch to Revit to Unreal Engine Workflow](https://www.linkedin.com/pulse/virtual-reality-architects-sketch-revit-unreal-engine-rajarshi-das/)
* [Game Architecture and Programming](https://www.slideshare.net/SumitJain39/game-architecture-and-programming)
