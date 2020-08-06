
---
title: 网卡硬件和ISO模型对应关系
author: "Joy Xu"
date: 2020-08-06 19:44:44
tags: [网卡, Linux Kernel]
---

一直不清楚MAC、PHY和Serdes之间到底是怎么连接的，今天理了理，
如果MAC没有内置PHY的话，可以认为MAC通过Serdes连接PHY。

# OSI模型

![OSI模型](/images/osi.png)

每层之间的关系如下图：

![OSI模型](/images/osi-ip.jpg)

# OSI模型和MAC、PHY之间的关系

MAC是链路层的协议，也可以表示IP，PHY是IP。
他们之间的关系可以参考下面这幅图
![OSI和MAC、PHY关系图](/images/osi-mac.jpg)

MAC和PHY之间可以通过Serdes互联。

一般在IP上，可以按照下面的方式互联：
![IP互联](/images/_10g-ethernet.gif)

# Linux Kernel驱动

在linux kernel中，网卡的驱动一般分成两部分MAC和PHY，也有MAC内置PHY的情况。
IP层上由kernel的tcp/ip协议栈实现。

MAC在kernel的结构体为struct net_device，主要处理收发包，以及IP中的ring buffer，
和kernel中的net框架打交道。

PHY一般通过MDIO总线进行访问。
PHY在kernel的结构体为struct phy_device，主要用来做自协商和link状态通知，有的PHY
也有高级功能，比如统计、Wake-on-LAN、MACSec等，它主要和内核中的phylib和phylink
框架打交道，PHY基本上已经非常标准化了，很多功能已经在phylib中实现了。

# 用户态工具

ethtool主要用来观察MAC，mii-tool基本已经不用，但是可以用它来观察PHY。

# 参考

* [The SERDES/transceiver design inside the Ethernet MAC controller](https://electronics.stackexchange.com/questions/467240/the-serdes-transceiver-design-inside-the-ethernet-mac-controller)
* [From the Ethernet MAC to the link partner](https://events19.linuxfoundation.org/wp-content/uploads/2017/12/chevallier-tenart-from-the-ethernet-mac-to-the-link-partner.pdf)
* [A Beginner’s Guide to Ethernet 802.3 Application Note](https://www.analog.com/media/en/technical-documentation/application-notes/EE-269.pdf)
* [TCP/IP vs. OSI: What’s the Difference Between the Two Models?](https://community.fs.com/blog/tcpip-vs-osi-whats-the-difference-between-the-two-models.html)
* [Altera's 10 Gbps Ethernet (XAUI) Solution](https://www.intel.com/content/www/us/en/programmable/solutions/technology/transceiver/protocols/pro-10gb_ethernet.html)
* [网口扫盲三:以太网芯片MAC和PHY的关系](https://www.cnblogs.com/jason-lu/p/3195473.html)
* [Octal 10/100/1000BASE-T PHY and 100BASE-FX/1000BASE-X SerDes with SGMII MAC Interface](https://www.microsemi.com/product-directory/gigabit-ethernet-phys/3910-vsc8658)
* [Tri-Mode Ethernet MAC v9.0](https://www.xilinx.com/support/documentation/ip_documentation/tri_mode_ethernet_mac/v9_0/pg051-tri-mode-eth-mac.pdf)
* [以太网MAC和PHY之间的接口总结](https://www.eda365.com/thread-280766-1-1.html)
