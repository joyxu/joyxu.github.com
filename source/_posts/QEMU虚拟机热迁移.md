---
title: QEMU虚拟机热迁移
author: Joy Xu
date: 2025-02-08 01:41:52
tags: [Linux Kernel, QEMU, VFIO]
---

# 什么是热迁移

虚拟机的迁移可以分为两大类: 离线迁移(Offline Migration)和热迁移(Live Migration)。
离线迁移是指在迁移之前, 将虚拟机暂停, 拷贝其状态到目的主机,在目的主机主机重新建立运行状态, 恢复执行。这种迁移技术适用于对服务可用性要求不高的场合。
热迁移, 是在保证虚拟机中服务正常运行的同时, 把虚拟机从一个物理主机拷贝到另一个物理主机。整个迁移过程对用户是透明的, 即用户感觉不到虚拟机位置的变化。

# QEMU虚机热迁移基本流程

QEMU虚拟机热迁移是指在host上QEMU命令行中使用`migrate`命令把指定的虚拟机从一个host迁移到另外一个host。
热迁移过程中会有短暂的停机时间，但目的是尽可能减少停机时间。
热迁移的性能好坏，一般会考虑3个指标：
* 整体迁移时间：越长越差
* 停机时间：越长越差
* 对业务性能的影响：迁移对于虚机中运行的服务的影响程度，可以通过每秒指令数来观察业务被中断的时间。

![热迁移完整流程](/images/qemu_live_migration_big_picture.png)

流程基本如下：

1. Setup

Start guest on destination, connect, enable dirty page logging and more

2. Transfer Memory
	* Guest continues to run
	* Bandwidth limitation (controlled by the user)
	* First transfer the whole memory
	* Iteratively transfer all dirty pages (pages that were written to by the guest).

3. Stop the guest
	* sync VM image(s) (guest’s hard drives).

4. Transfer State
	* As fast as possible (no bandwidth limitation)
	* All VM devices’ state and dirty pages yet to be transferred

5. Continue the guest
	* On destination upon success,broadcast “I’m over here” Ethernet packet to announce new location of NIC(s).
	* On source upon failure (with one exception).

# QEMU虚机热迁移实现

热迁移关键stage是上图中的stage2、3和5,QEMU中对应的实现如下：

1. 将虚拟机所有RAM pages设置成dirty，主要函数`ram_save_setup`
2. 持续迭代将虚拟机的dirty pages发送到dst，直到达到一定条件，比如dirty pages数量比较少, 主要涉及函数`migration_thread`中的whille循环中的`migration_iteration_run`以及内存保存函数`ram_save_iterate`
3. 停止src上面的guest，把剩下的dirty pages发送到dst，之后发送设备状态，主要函数`qemu_savevm_state_complete_precopy`

其中1和2是上图中的灰色区域，3是灰色和左边的区域。
之后就可以在dst上面继续运行qemu程序了。

## 源端的关键实现

	migration_thread
	 qemu_savevm_state_setup //标记所有RAM pages为dirty
	  ram_save_setup[save_setup]
	   ram_init_all
	    ram_init_bitmaps
	     ram_list_init_bitmaps
	      bitmap_newjiangp
	      bitmap_set

	migration_iteration_run //快速迭代拷贝脏内存，当剩余脏页量小于bandwidht * downtime时，进入停机拷贝阶段，参数可以参考hmp_info_migrate
	 qemu_savevm_state_pending
	  ram_save_pending[save_live_pending] //确定还要传输的字节数
	 qemu_savevm_state_iterate
	  ram_save_iterate[save_live_iterate] //把dirty pages传到dst上面
	   ram_find_and_save_block
	    send //调用具体的传输方法，比如TCP、RDMA等

	migration_completion
	 vm_stop_force_state
	 qemu_savevm_state_complete_precopy
	  qemu_savevm_state_complete_precopy_iterable
	   ram_save_complete[save_live_complete_precopy]
	    ram_find_and_save_block //拷贝最后的脏内存

## 目的端的关键实现

	migration_incoming_process
	 qemu_loadvm_state 
	  qemu_loadvm_section_start_full
	   find_se //处理源端发过来的各个section
	   vmstate_load
	    ram_load[load_state] //接收到的数据拷贝到目的端虚拟机的内存上
	     ram_load_precopy
	      qemu_get_buffer

## QEMU VDPA设备热迁移

vdpa当前支持virtio-net，这块的迁移流程已经很成熟，因为virito设备的规范已经有virtio spec覆盖。

![VDPA直通设备热迁移完整流程](/images/qemu_live_migration_big_picture_vdpa.png)

## VFIO设备热迁移

VFIO直通设备在热迁移过程中，首先要求设备停流排空，把流量收敛住，进入一个稳态，过程中已经在路上的流量就涉及到DMA内存标脏，进入稳态之后，设备需要保存和恢复状态。

在热迁移过程中涉及到很多回调，QEMU中主要涉及到`SaveVMHandlers`结构体，针对内存的回调基本在`savevm_ram_handlers`中。
那如果虚机中有VFIO直通设备，同样也需要实现针对VFIO设备的`savevm_vfio_handlers`回调, 另外也要实现`qdev_add_vm_change_state_handler_full` 和 `migration_add_notifier` 这两个回调。

* `SaveVMHandlers` 是数据面，它定义了设备状态如何被持久化传输，没有它，设备状态就无法迁移，解决 “我的状态是什么？怎么存？怎么读？” 的问题。
* `qdev_add_vm_change_state_handler_full` 是控制面，它允许设备在迁移过程的关键节点（开始前、完成后、失败时、加载前后等）执行管理操作，确保设备在迁移前后行为正确，但这些操作本身不产生迁移流数据。它调用 SaveVMHandlers 来传输实际状态。解决 “VM 要暂停/恢复了，我需要做点啥准备/收尾？” 的问题（特别是与迁移相关的暂停/恢复）。
* `migration_add_notifier` 是错误处理，它提供系统级别对迁移状态变化的感知，针对VFIO设备，主要执行设备状态回滚操作。解决 “迁移这个事儿本身现在怎么样了？（开始了吗？成功了吗？失败了吗？）” 的问题。

内核态的vfio厂商驱动中，也要对应实现上面的ioctl，以及migration序列化结构体，结构体中除了保存寄存器以外，还有硬件上不是通过寄存器呈现的硬件状态，比如通过mailbox下发给硬件的DMA地址。

![VFIO直通设备热迁移完整流程](/images/qemu_live_migration_big_picture_vfio.png)

![VFIO直通设备热迁移完整流程2](/images/qemu_live_migration_big_picture_vfio2.png)

在QEMU的官网上，针对VFIO设备迁移在QEMU内的实现，专门有一段描述:(括号中的状态，分别代表 VM状态，迁移状态，VFIO设备状态)

![VFIO直通设备热迁移QEMU流程](/images/qemu_live_migration_vfio_qemu.png)

### intel E810网卡的VF热迁移实际流程

![intel E810 VFIO直通设备热迁移完整流程](/images/qemu_live_migration_vfio_e810.png)

### 停机时间优化-通过pre-copy减少停机时间

由于设备的状态保存和恢复发生在停机阶段，为了尽可能减少虚机停机时间，也会考虑pre-copy。

![直通设备热迁移流程](/images/qemu_live_migration_vfio_qemu2.png)

### VFIO设备迁移相关patchset

VFIO热迁移的历史可以追踪一下patch set:
* [vfio: Define device migration protocol v2](https://patchwork.kernel.org/project/netdevbpf/patch/20220220095716.153757-10-yishaih@nvidia.com/#24749543)
* [vfio: VFIO migration support with vIOMMU](https://lore.kernel.org/all/20230622214845.3980-1-joao.m.martins@oracle.com/)
* [Multifd: device state transfer support with VFIO consumer](https://lore.kernel.org/all/cover.1738171076.git.maciej.szmigiero@oracle.com/)

内核态的驱动适配可以参考这个patch set:
* [vfio/hisilicon: add ACC live migration driver](https://patchwork.kernel.org/project/linux-pci/patch/20220308184902.2242-1-shameerali.kolothum.thodi@huawei.com/)

# 未来演进

* ARM架构特性演进，比如在iommu侧也实现mmu侧的相关特性，比如：FEAT_TLBIRANG减少TLBI次数，FEAT_BBM=2, MMU/IOMMU硬件标脏等。

# 参考

* [Live Migrating VFIO, vhost-user,and vfio-user Devices](https://kvm-forum.qemu.org/2021/Live%20Migrating%20VFIO%2C%20vhost-user%2C%20and%20vfio-user%20Devices.pdf)
* [Live Migration of Virtual Machines](https://dl.acm.org/doi/abs/10.5555/1251203.1251223#core-collateral-purchase-access)
* [Live Migrating QEMU-KVM Virtual Machines](https://developers.redhat.com/blog/2015/03/24/live-migrating-qemu-kvm-virtual-machines)
* [论文笔记 Live Migration of Virtual Machines NSDI, 2005](https://www.cnblogs.com/yuquanlaobo/archive/2013/01/17/2863040.html)
* [qemu热迁移简介](https://terenceli.github.io/%E6%8A%80%E6%9C%AF/2018/03/01/qemu-live-migration)
* [Journey of advancing device migration for virtio PCI hardware devices](https://netdevconf.info/0x18/docs/netdev-0x18-paper22-talk-paper.pdf)
* [Journey of advancing virtio live migration](https://netdevconf.info/0x18/docs/netdev-0x18-paper22-talk-slides/virtio-live-migratation-slides.pdf)
* [VFIO device migration](https://www.qemu.org/docs/master/devel/migration/vfio.html)
* [A Perfect Solution for Live Migration with Pass-through Devices by Quan Xu](https://liujunming.top/2022/05/21/A-Perfect-Solution-for-Live-Migration-with-Pass-through-Devices-by-Quan-Xu/)
* [基于E810网卡的VF热迁移](https://liujunming.top/2023/10/05/%E5%9F%BA%E4%BA%8EE810%E7%BD%91%E5%8D%A1%E7%9A%84VF%E7%83%AD%E8%BF%81%E7%A7%BB/)
* [VM 热迁移详解](https://cshuo.top/2016/09/10/live_migration/)
* [Unleashing VFIO's Potential: Code Refactoring and New Frontiers in Device Virtualization](https://kvm-forum.qemu.org/2024/KVM_Forum_2024_-_VFIO_5LSTtyJ.pdf)
* [QEMU live migration device state transfer parallelization via multifd channels](https://kvm-forum.qemu.org/2024/kvm-forum-2024-multifd-device-state-transfer_3K5EQIG.pdf)
* [virtio-net: add support for SR-IOV emulation](https://kvm-forum.qemu.org/2024/Unleashing_SR-IOV_on_Virtual_Machines_qSX9OJ9.pdf)
* [IDPF Live Migration Support](https://netdevconf.info/0x17/docs/netdev-0x17-paper30-talk-slides/idpf_live_migration_support.pdf)
* [Hardware Friendly Vhost vDPA Towards an Efficient and Migratable Device Model](https://www.patchew.org/2022/KVM22-Migratable-Vhost-vDPA.pdf/)
* [vDPA-net live migration with Shadow Virtqueue](https://kvm-forum.qemu.org/2023/vDPA_sw_lm_-_KVM2023_6Ix6R5i.pdf)
* [Live Migration with Mediated Device](https://www.linux-kvm.org/images/f/fa/Live_migration_with_mdev_device_-_2017_0.pdf)
* [vDPA Live Migration Downtime improvements for net devices](https://netdevconf.info/0x18/sessions/talk/vdpa-live-migration-downtime-improvements-for-net-devices.html)
* [SW Assisted vDPA for Live Migration](https://www.dpdk.org/wp-content/uploads/sites/35/2018/12/XiaoWang-DPDK-US-Summit-SW-assisted-VDPA-for-LM-v2.pdf)
