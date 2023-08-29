---
title: ARM服务器软件栈系列1
author: Joy Xu
date: 2023-08-29 01:56:06
tags: [Linux Kernel, ARM Server]
---

# 目的

无意在ARM官网上看到了N2的参考设计文档，一眼就被惊艳了。主要有两点：

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

	![软件架构图](/images/arm_server_software_stack.png)

# AP和协处理器之间的通信协议

在N2的架构中，SCP处理器用来做启动、复位、时钟和电源管理;MCP处理器用来做RAS，并和
带外的BMC通信。SCP和MCP均是ARM M7 core。
SCP、MCP和AP之间一般通过SCMI协议进行通信，SCMI通信的基地址一般通过ACPI的PCCT(platform communication table)呈现给AP,
PCCT的具体定义可以在ACPI规范的14章找到，示例如下图：

	![SCMI PCCT](/images/arm_server_scmi.png)

结合ARM 参考软件栈，流程如下图(引用自Power and Performance Management using Arm SCMI Specification)：

	![软件架构图](/images/arm_server_scmi_software_stack.png)


	![软件架构图](/images/arm_server_software_stack.png)
	![软件架构图](/images/arm_server_software_stack.png)
	![软件架构图](/images/arm_server_software_stack.png)

# 参考

* [Arm Neoverse N2 reference design Technical Overview](https://developer.arm.com/documentation/102337/0000/Software-stack/About-the-software?lang=en)
* [Neoverse Reference Design Platform Software](https://neoverse-reference-design.docs.arm.com/en/latest/readme.html)
* [The Forbidden Arm Server that is Banned in the US](https://www.servethehome.com/the-forbidden-arm-server-that-is-banned-in-the-us/6/)
* [AST2500 NC-SI功能调试](https://blog.csdn.net/zhaoxinfan/article/details/82890449)
* [System Control & Management Interface](https://static.linaro.org/connect/lvc20/presentations/LVC20-119-0.pdf)
* [Power and Performance Management using Arm SCMI Specification](https://developer.arm.com/documentation/102886/001?lang=en)
