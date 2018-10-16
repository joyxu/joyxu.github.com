title: integrate avocado with lava
author: Joy Xu
date: 2017-11-16 14:36:53
tags: [Virtualization, avocado, avocado-vt]
---

# 目的

本文主要记录下集成lava和avocado的思路，趟过的坑和后续的TODO。

# 前言

虚拟化测试或者说qemu/kvm的测试，涉及到太多的变量，比如qemu设备类型、core的个数、
内存大小和kernel版本等等，组合条件实在是太多。
想追求一个覆盖全集的基线测试集合，短时间内搞定实在不现实，只能看看业界标杆是怎么做的了。

从几家主流OS发型商和社区来看，avocado/avocado-vt绝对是主流：
* SUSE: [https://build.opensuse.org/package/show/Virtualization:Tests/avocado](https://build.opensuse.org/package/show/Virtualization:Tests/avocado)
* Redhat: [https://fedoraproject.org/wiki/Testing_KVM_with_kvm_autotest](https://fedoraproject.org/wiki/Testing_KVM_with_kvm_autotest)
* KVM: [http://www.linux-kvm.org/page/KVM-Autotest](http://www.linux-kvm.org/page/KVM-Autotest)

而且Redhat还把它开源了，有:
* [代码](https://github.com/avocado-framework/avocado-vt)
* [文档](http://avocado-vt.readthedocs.io/en/latest/)
* [邮件列表](https://www.redhat.com/mailman/listinfo/avocado-devel)
* 甚至还有[开发计划](https://trello.com/b/WbqPNl2S/avocado)，简直良心极了!

再看看它的宣讲，有水分但也实在。
![why-avocado](/images/why-avocado.png)

于是赶紧抓代码下来，在我们的单板开始实验了。

# avocado和avocado-vt是什么和能做什么？

官方的文档实在是TL;DR。
简单来讲，avocado是基于autotest优化的一个测试框架，它可以当做一个app在本地执行，也可以搭成服务器和客户端的模式运行；
avocado-vt是avocado的一个专门针对虚拟化的插件，它可以控制qemu/libvirt/spice的输入和输出，
还可以根据配置条件组合成不同配置的虚拟机，从虚拟化测试用例仓库按照配置提取并执行相应的测试用例。

看看下面这个图，能这么组合参数的话，就知道用处有多大了：）
![笛卡尔范例](/images/multiplexer.png)

# 如何集成到LAVA？
既然avocado/avocado-vt可以单独作为一个应用执行的话，集成到LAVA就和集成普通的shell没什么区别了，
只需要安装依赖的库，把代码下载下来编译执行就可以了，
另外还要注意下如果要写成自动化部署的话，github或者某些代码仓库下载速度慢的话，可以放到本地文件服务器上。

# 都趟过哪些坑

## 添加ubuntu的arm64支持
avocado是Redhat主导的，目前[网上有Redhat的支持](https://github.com/huangwei/linaro-sfo17/blob/master/test-qemu-rhel.cfg),
但是我们平时测试的host和比较容易下载到的rootfs主要还是ubuntu，只能自己撸起袖子加了，
这里可以[参考下我的commit](https://github.com/joyxu/avocado-vt/commit/2b93fdb7af9eec0394aa247056226270cca1733ea)

另外记得下载带有grub的ubuntu cloud镜像，具体原因可以看戳[这个链接](https://lists.ubuntu.com/archives/ubuntu-cloud/2013-December/000929.html)。

## qemu-kvm无法找到grub和ubuntu的rootfs
这个真的很坑了，avocado启动arm的virt虚拟机需要使用`AAVMF_CODE.fd`和`AAVMF_VARS.fd`，而使用virtio-blk-pci的话，就是不自动启动
镜像硬盘下的grub.efi，没办法只能把virtio-blk-pci先禁用了。
另外使用virtio-scsi-pci的时候，能找到grub和内核,可是又挂不上ubuntu的rootfs，看了下代码，需要加上"disable-modern=off,disable-legacy=on"。

## 使能ubuntu cloud镜像的root账号和运行远程ssh用户登录
网上有很多种使能ubuntu cloud镜像root用户的方法，但是[这个是最简单有效的](https://askubuntu.com/questions/92556/how-do-i-boot-into-a-root-shell)。
有了root账号之后，修改sshdconfig就好了。

## 换内核
ubuntu cloud镜像中已经有了内核，但是我们希望使用我们自己的内核怎么办呢？可以参考[这个脚本](https://github.com/joyxu/linaro-test-definitions/blob/master/ubuntu/scripts/qemu-kvm/test.sh)，
在host内核中使能nbd模块后，使用qmeu-nbd挂载ubuntu镜像，把自己的内核替换掉就好了。

# ToDo
* 使能其它配置选项，比如vhost、tap、macvlan、macvtap、virtio-blk-pci和virtio-mmio等等。
* 分析当前失败的测试用例，自动测试结果在[这里](http://120.31.149.194:800/dashboard/image-charts/Plinth)
* 使能libvirt的自动测试

# 参考文档
1 [Avocado Auto Testing for AArch64 Virtualization](https://www.slideshare.net/linaroorg/avocado-auto-testing-for-aarch64-virtualization-sfo17502)
2 [AVOCADO AND JENKINS](https://schd.ws/hosted_files/devconfcz2016/c8/devconf_yash_lukas_final.pdf)
