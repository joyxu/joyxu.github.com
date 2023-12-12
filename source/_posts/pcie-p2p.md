---
title: pcie及p2p
author: Joy Xu
date: 2023-12-11 02:15:15
tags: [linux kernel, pcie, p2p]
---

# 综述

P2P是PCIe协议定义的三种数据传输方式之一(PIO，P2P和DMA)。

PCIe P2P是指两个PCIe设备直接通信，通信的数据不经过CPU处理从一个PCIe设备交换到另外一个PCIe设备。
因此有两个隐藏的前提条件：

* 至少某一个PCIe设备拥有内存，而不是简单的一个PCIe设备去访问另外一个PCIe设备的BAR空间或者寄存器
* 至少某一个PCIe设备拥有算力

# PCIe设备枚举建链

## PCI设备的地址空间

PCI协议定义了三种地址空间：mmio地址空间(memory address space)，io地址空间(io address space)和配置空间(configure address space)。
CPU要访问前面两种地址空间分别通过mmio和pio,而对于配置空间则使用io端口 CF8/CFC或者ECAM方式访问，在arm64平台上只支持ECAM机制。

## ECAM机制

ECAM其实也是基于mmio访问的，以ACPI启动举例，ACPI的MCFG(memory-maped configuration)表格中会配置ECAM的基地址。

![pcie ecam](/images/pcie_ecam_mcfg.png)

完整流程如下：

![pcie ecam_overview](/images/pcie_ecam_overview.png)

## 设备枚举建链接

建立好配置空间之后，后面就是读取配置空间信息，建立pcie设备树，具体流程如下:



# 参考

* [PCIe扫盲系列博文连载目录篇](http://blog.chinaaet.com/justlxy/p/5100053328)
* [PCIe扫盲——PCI总线的三种传输模式](http://blog.chinaaet.com/justlxy/p/5100053095)
* [PCIe ECAM机制](https://blog.csdn.net/u013253075/article/details/130755162)
* [Linux源码阅读——PCI总线驱动代码（三）PCI设备枚举过程](https://blog.csdn.net/u013253075/article/details/123301127)
* [A Practical Tutorial on PCIe for Total Beginners on Windows (Part 1)](https://ctf.re/windows/kernel/pcie/tutorial/2023/02/14/pcie-part-1/)
* [UEFI——PCIe子系统(I)](https://blog.csdn.net/weixin_43921686/article/details/132136732)
