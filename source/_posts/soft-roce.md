
---
title: 使用QEMU测试soft roce
author: "Joy Xu"
date: 2020-12-24 19:44:44
tags: [RoCE, Linux Kernel]
---

# kernel config

打开以下选项

	CONFIG_INFINIBAND=y
	CONFIG_INFINIBAND_ADDR_TRANS=y
	CONFIG_INFINIBAND_ADDR_TRANS_CONFIGFS=y
	CONFIG_RDMA_RXE=y

# QEMU的rootfs依赖包

依赖以下包：

* iproute2,提供rdma工具，支持rdma link add命令
* ibverbs-providers，提供librxe_rdma.so
* perftest，提供ib_send_bw等测试工具

# QEMU命令

	qemu-system-aarch64 -s \
	-machine virt \
	-cpu cortex-a57 \
	-smp 2 \
	-m 4096 \
	-kernel $KERNEL \
	-initrd ${INITRD} \
	-rtc base=localtime \
	-netdev user,id=user0,hostfwd=tcp::5000-:22 \
	-device virtio-net-pci,netdev=user0 \
	-append 'nokaslr console=ttyAMA0 earlycon kmemleak=on' \
	-nographic

 虚拟机启动之后，运行以下命令：

	 rdma link add rxe_eth0 type rxe netdev eth0
	 rdma link
	 mkdir -p /usr/local/etc/libibverbs.d/
	 echo "driver rxe" > /usr/local/etc/libibverbs.d/rxe.driver
	 ib_send_bw -d rxe_eth0 -n 5 -s 100 -Q 1 &
	 ib_send_bw -d rxe_eth0 -n 5 -s 100 -Q 1 127.0.0.1

# 实例

![soft RDMA运行实例](/images/soft_rdma.gif)
![soft RDMA调试实例](/images/soft_roce_debug.PNG)

# 参考

* [RDMA API 应用](https://zhuanlan.zhihu.com/p/481256797)
* [RDMA-tutorial](https://github.com/StarryVae/RDMA-tutorial)
* [RDMA 高级](https://zhuanlan.zhihu.com/p/567720023)
* [RDMA之Memory Region](https://zhuanlan.zhihu.com/p/156975042)
* [用户程序调用 IBVerbs](https://gwzlchn.github.io/202208/rdma-stack-02/)
* [ibv_reg_mr](https://www.rdmamojo.com/2012/09/07/ibv_reg_mr/)
