---
title: ARM服务器软件栈系列1
author: Joy Xu
date: 2023-08-29 01:56:06
tags: [Linux Kernel, ARM Server]
---

# 目的

无意在ARM官网上看到了[N2的参考设计文档](https://developer.arm.com/documentation/102337/0000/Software-stack/About-the-software?lang=en)，一眼就被惊艳了。主要有两点：

* 这个文档的视野非常全面，作为一个参考架构设计文档，值得推荐和学习
* 这个文档不仅公开了软件设计，也公开了硬件设计，范围不只覆盖了AP，还覆盖了功耗core，RAS core，非常全面了

# 服务器硬件形态

N2的硬件架构图就是一个典型的服务器硬件架构，加上BMC和CPLD部分就更好了

![N2硬件架构图](/images/arm_server_hardware_topo.png)

下面是一张带BMC和CPLD的图

![硬件架构图](/images/arm_server_hardware_topo2.png)

BMC的功能以及和AP之间的交互如下图

![硬件架构图-BMC](/images/arm_server_bmc.jpg)

# 服务器软件形态

软件形态主要是解释各处理器的职责。

![软件架构图](/images/arm_server_software_stack.png)

在N2的架构中，SCP处理器用来做启动、复位、时钟和电源管理。
MCP处理器用来做RAS，并和带外的BMC通信。SCP和MCP均是ARM M7 core。
AP处理器运行Arm Trust Firmware、UEFI和Linux Kernel操作系统。


# AP和协处理器之间的通信协议

SCP、MCP和AP之间一般通过SCMI协议进行通信。
SCMI通信的基地址一般通过ACPI的PCCT(platform communication table)呈现给AP,PCCT的具体定义可以在ACPI规范的14章找到，示例如下图：

![SCMI PCCT](/images/arm_server_scmi.png)

# 电源管理-SCP

结合ARM 参考软件栈，流程如下图(引用自Power and Performance Management using Arm SCMI Specification)：

![软件架构图](/images/arm_server_scmi_software_stack.png)

# RAS处理-MCP

ARM定义了一套RAS处理机制SDEI，一旦发生了RAS错误，Firmware会通知Linux Kernel。
但是通常来讲如果有了RAS专用处理核，一般都走ACPI的APEI机制。

![SDEI RAS机制](/images/arm_server_ras_sdei.png)

# 带外管理-BMC

服务器上一般都有BMC板级管理芯片，BMC主要用来管理风扇，电源，固件升级，远程控制等。

![BMC funciton view](/images/arm_server_bmc3.png)

AP和BMC一般有LPC、USB、PCIe、SMBUS总线等。PCIe一般用于KVM(键盘、鼠标和显示的重定向);
USB多用于虚拟磁盘，通过它支持光盘、ISO镜像用于安装操作系统。
BMC和AP之间的接口叫作system interface，简称SI，常见的SI有KCS、SMIC、BT和SSIF传输协议，这些协议均已被Linux Kernel主线支持，
驱动在 `drivers/char/ipmi` 中。

![BMC logic view](/images/arm_server_bmc2.png)

# 参考

* [Arm Neoverse N2 reference design Technical Overview](https://developer.arm.com/documentation/102337/0000/Software-stack/About-the-software?lang=en)
* [Neoverse Reference Design Platform Software](https://neoverse-reference-design.docs.arm.com/en/latest/readme.html)
* [The Forbidden Arm Server that is Banned in the US](https://www.servethehome.com/the-forbidden-arm-server-that-is-banned-in-the-us/6/)
* [AST2500 NC-SI功能调试](https://blog.csdn.net/zhaoxinfan/article/details/82890449)
* [System Control & Management Interface](https://static.linaro.org/connect/lvc20/presentations/LVC20-119-0.pdf)
* [Power and Performance Management using Arm SCMI Specification](https://developer.arm.com/documentation/102886/001?lang=en)
* [使用SDEI上报RAS故障](https://blog.csdn.net/jingr1/article/details/126216896?spm=1001.2014.3001.5501)
* [Explaining the Baseboard Management Controller or BMC in Servers](https://www.servethehome.com/explaining-the-baseboard-management-controller-or-bmc-in-servers/)
* [IPMI Basics](https://www.thomas-krenn.com/en/wiki/IPMI_Basics)
* [x86服务器BMC基板管理控制器介绍](https://www.cnblogs.com/zhangxinglong/p/13292092.html)
* [BMC常见接口协议](https://www.ctyun.cn/developer/article/445761300189253)
