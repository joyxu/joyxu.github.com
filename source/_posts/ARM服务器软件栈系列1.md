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

N2的硬件架构图就是一个典型的服务器硬件架构，一般板级形态上还有BMC、CPLD等协同。

![N2硬件架构图](/images/arm_server_hardware_topo.png)

上图中涉及到的ARM IP均可以从[ARM官网](https://developer.arm.com/developer/ip-products/)获取，从而拼成一个SoC，不同SoC间互联之后，同时加上板级设计，进而形成服务器单板硬件。

![ARM ipexplorer](/images/arm_server_ip_explorer.png)

ARM官网专门有一个check list文档[Arm Design Checklists User Guide](https://developer.arm.com/documentation/110244/0100/Arm-Design-Checklists)，帮助SoC设计者检验。

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

服务器一般还带一个BMC芯片做带外的板级管理，BMC主要用来管理风扇，电源，固件升级，远程控制等，具体功能如下图

![硬件架构图-BMC](/images/arm_server_bmc.jpg)

涉及到的协议，总线和功能一般如下：

![BMC funciton view](/images/arm_server_bmc3.png)

AP和BMC一般有LPC、USB、PCIe、SMBUS总线等。PCIe一般用于KVM(键盘、鼠标和显示的重定向);
USB多用于虚拟磁盘，通过它支持光盘、ISO镜像用于安装操作系统。
BMC和AP之间的接口叫作system interface，简称SI，常见的SI有KCS、SMIC、BT和SSIF传输协议，这些协议均已被Linux Kernel主线支持，

![BMC logic view](/images/arm_server_bmc2.jpg)

以Amper为例，实际上板级结构如下图

![BMC real case](/images/arm_server_hardware_topo2.png)

内核的驱动在 `drivers/char/ipmi` 中:
`ipmi_ssif.ko`: 支持通过SMBUS接口和发送消息
`ipmi_si.ko`: 对应system interface，一般都使用该驱动，支持platform、acpi、SMBIOS(DMI)和PCI
`ipmi_msghandler.ko`:实现IPMI协议
`ipmi_devinf.ko`: 对外呈现ipmi设备，比如`/dev/ipmi0`,提供设备的IOCTL操作

## BMC virtual-meida 安装系统

BMC一般提供一个网络界面，这个界面上用户可以上传一个ISO，远程的服务器可以读取该ISO来安装系统。
这个机制一般通过如下机制实现[参考openBMC实现](https://github.com/openbmc/docs/blob/master/designs/virtual-media.md)

![BMC virtual media](/images/arm_server_bmc_virtual_media.png)

# 可信度量启动

如果要支持可信度量启动，一般还会加上TF-M(PSA Firmware)。
如果还要支持运行时安全(runtime security subsystem)，一般会在TF-M中加上RSS，在N2的参考设计中，这是一个[M55的core](https://neoverse-reference-design.docs.arm.com/en/latest/platforms/rdfremont/docs/rss.html)。

加上可信度量和运行时安全core之后，整个启动过程如下图:

![TF-M & RSS](/images/arm_server_tfm_rss.png)

## 二进制构成和Flash layout

TF-M分成三部分：TF-M BL11, BL12, BL2。其中BL11是出厂时烧在TF-M芯片的ROM中，其它在flash里面。

Flash的构成可以参考[N2 TF-M BL2的代码](https://gitlab.arm.com/infra-solutions/reference-design/platsw/trusted-firmware-m/-/blob/refinfra-fremont/platform/ext/target/arm/rss/rdfremont/bl2/flash_map_bl2.c)
或者[flash_layout.c](https://gitlab.arm.com/infra-solutions/reference-design/platsw/trusted-firmware-m/-/blob/refinfra-fremont/platform/ext/target/arm/rss/rdfremont/flash_layout.h#L24)

		/* Flash layout on RSS with BL2 (multiple image boot):
		 *
		 * 0x3100_0000 BL2 - MCUBoot (64 KB)
		 * 0x3101_0000 BL2 - MCUBoot (64 KB)
		 * 0x3102_0000 Secure image     primary slot (384 KB)
		 * 0x3108_0000 Non-secure image primary slot (384 KB)
		 * 0x310E_0000 Secure image     secondary slot (384 KB)
		 * 0x3114_0000 Non-secure image secondary slot (384 KB)
		 * 0x311A_0000 SCP BL1 primary slot (512 KB)
		 * 0x3122_0000 SCP BL1 secondary slot (512 KB)
		 * 0x312A_0000 MCP BL1 primary slot (512 KB)
		 * 0x3132_0000 MCP BL1 secondary slot (512 KB)
		 * 0x313A_0000 LCP BL1 primary slot (64 KB)
		 * 0x313B_0000 LCP BL1 secondary slot (64 KB)
		 * 0x312A_0000 AP BL1 primary slot (512 KB)
		 * 0x3132_0000 AP BL1 secondary slot (512 KB)
		 */


把不同二进制打包成一个二进制，可以参考[fiptool这个工具](https://gitlab.arm.com/infra-solutions/reference-design/platsw/trusted-firmware-m/-/blob/refinfra-fremont/docs/platform/arm/rss/readme.rst)

		fiptool create \
				--align 8192 --rss-bl2           bl2_signed.bin \
				--align 8192 --rss-ns            tfm_ns.bin \
				--align 8192 --rss-s             tfm_s.bin \
				--align 8192 --rss-sic-tables-ns tfm_ns_sic_tables_signed.bin \
				--align 8192 --rss-sic-tables-s  tfm_s_sic_tables_signed.bin \
				--align 8192 --rss-scp-bl1       <signed Host SCP BL1 image> \
				--align 8192 --rss-ap-bl1        <signed Host AP BL1 image> \
				fip.bin


以SCP二进制为例，在flash上的布局如下：

![flash scp sample](/images/arm_server_flash_scp.png)

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
* [IPMI的几个问题](https://www.cnblogs.com/klb561/p/9070001.html)
* [IPMI2：ipmi逻辑设备](https://blog.csdn.net/qq_34160841/article/details/121728388)
* [BMC virtual media](https://github.com/openbmc/docs/blob/master/designs/virtual-media.md)
* [服务器BMC与IPMI基础知识](https://blog.csdn.net/star871016/article/details/112257689)
* [Ampere Altra 64-Bit Multi-Core Processor Platform Hardware Design Specification](https://connect-admin.amperecomputing.com/api/secure-file-download/download-regular/?file=Altra_Platform_HW_Design_Specification_v1_12_20230110_7fec8e8c20.pdf&type=technical-document&doc_id=437)
* [Arm Corstone-1000](https://corstone1000.docs.arm.com/en/latest/software-architecture.html)
* [Arm Neoverse N2 Automotive Reference Stack Documentation](https://rd-n2-automotive.docs.arm.com/en/v1.0/design/boot_process.html)
* [SoC 设计流程](https://blog.csdn.net/ygyglg/article/details/131159942)
