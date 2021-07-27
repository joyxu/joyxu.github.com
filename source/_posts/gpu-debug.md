---
title: GPU 调试
author: Joy Xu
date: 2021-06-21 07:01:21
tags: [linux, gpu]
---

## Renderdoc

Renderdoc是最受欢迎的开源工具，不仅被大多数硬件厂商支持，也被大的游戏引擎厂商支持。
使用起来也很简单，texture view可以看贴图的问题，mesh view看模型的问题。

### 参考

* [Graphics Debugging using RenderDoc](https://matiaslavik.wordpress.com/2020/01/17/graphics-debugging-using-renderdoc/)
* [Debugging Tools](https://www.khronos.org/opengl/wiki/Debugging_Tools)
* [性能分析工具集](https://blog.csdn.net/linjf520/article/details/114688952?spm=1001.2014.3001.5501)

## 性能测试

### drawoverhead

piglit测试套中的`src/perf/drawoverhead`可以单独编译成可执行程序，用来测试性能。

### 参考

* [Enhnace your pipe](https://www.supergoodcode.com/layin-that-pipe/)
