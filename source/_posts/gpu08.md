---
title: Linux GPU系列-GPU驱动到底做什么
author: Joy Xu
date: 2021-05-21 03:18:14
tags: [linux, gpu]
---

GPU驱动和其他设备驱动其实差异不大，主要就两个面：

* 控制面：设置state，通过fence做同步等
* 数据面：把数据从CPU搬到GPU，这里主要涉及到数据格式和数据layout，类似网卡的描述符，以及设备内存的管理。


## 数据面

本文以OpenGL为例，主要讲数据如何搬到GPU的。

![cpu_gpu_data](/images/cpu_gpu_data1.png)

![cpu_gpu_data](/images/cpu_gpu_data2.png)

![cpu_gpu_data](/images/cpu_gpu_data3.png)

![cpu_gpu_data](/images/cpu_gpu_data4.png)

![cpu_gpu_data](/images/cpu_gpu_data5.png)

![cpu_gpu_data](/images/cpu_gpu_data6.png)

![cpu_gpu_data](/images/cpu_gpu_data7.png)

![cpu_gpu_data](/images/cpu_gpu_data8.png)

![cpu_gpu_data](/images/cpu_gpu_data9.png)

![cpu_gpu_data](/images/cpu_gpu_data10.png)

![cpu_gpu_data](/images/cpu_gpu_data11.png)

![cpu_gpu_data](/images/cpu_gpu_data12.png)

![cpu_gpu_data](/images/cpu_gpu_data13.png)

![cpu_gpu_data](/images/cpu_gpu_data14.png)

![cpu_gpu_data](/images/cpu_gpu_data15.png)

![cpu_gpu_data](/images/cpu_gpu_data16.png)

![cpu_gpu_data](/images/cpu_gpu_data17.png)

![cpu_gpu_data](/images/cpu_gpu_data18.png)

## 控制面

Todo

## 参考

[cpu-gpu](https://chamilo.grenoble-inp.fr/main/document/document.php?cidReq=ENSIMAG4MMG3D6&id_session=0&gidReq=0&gradebook=0&origin=&action=download&id=1109723)
