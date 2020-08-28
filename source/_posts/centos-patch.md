---
title: Centos内核
author: Joy Xu
date: 2020-08-28 19:18
tags: [Linux Kernel]
---

# CentOS版本

CentOS目前有3个版本，6,7和8。
具体参考 [官网](https://wiki.centos.org/FAQ/General#Why_is_the_CentOS_Project_taking_over_my_Website.3F)
各个版本的生命周期、内核版本可以参考 [官网的详细介绍](https://wiki.centos.org/About/Product)

# CentOS内核代码

CentOS内核config：https://git.centos.org/rpms/kernel/releases
SRPM包归档：http://vault.centos.org/

比如最新8的内核SRPM包：http://vault.centos.org/8.2.2004/BaseOS/Source/SPackages/

下载下来之后，通过rpm2cpio转成CPIO，再用cpio解压就可以拿到内核的压缩包，具体步骤：

	rpm2cpio kernel-4.18.0-193.el8.src.rpm | cpio -idmv
	tar xJf linux-4.18.0-193.el8.tar.xz

但是注意的是，里面是没有任何git信息的。
另外还有一个有意思的是里面解压出来会有一个check-kabi的脚本。


