---
title: Ubuntu最小GUI文件系统
author: Joy Xu
date: 2021-11-02 01:04:08
tags: [Ubuntu, 最小图形界面, MESA, piglit, Virtio GPU, GPU, QEMU, aarch64]
---

# 目的

以前做小文件系统的时候，都是基于Busybox或者BuildRoot等工具，但是再做GPU虚拟化
过程中，由于图形软件堆栈依赖太多，很难把全部的包都包含到小文件系统中。这篇文章
介绍如何基于Ubuntu官方提供的base文件系统加入最小图形软件栈。


# Ubuntu最小GUI文件系统的具体步骤：

我的系统是Ubuntu20.04的，为了方便共享host和guest，guest也选择Ubuntu20.04的base包。
有两种方式生成base包：

* 可以直接从`http://cdimage.ubuntu.com/ubuntu-base/releases/`里面下载
* 或者通过qemu-debootstrap下载，如果host和guest是同一种arch，效果和debootstrap一样

		sudo apt-get install debootstrap qemu-user-static schroot
		dd if=/dev/zero of=./rootfs.img bs=1M count=4000
		mke2fs -t ext4 ./rootfs.img
		mkdir ./test
		sudo mount -o loop ./rootfs.img ./test

		#option 1
		wget -c http://cdimage.ubuntu.com/ubuntu-base/releases/20.04/release/ubuntu-base-20.04.3-base-arm64.tar.gz
		sudo tar -xzvf ubuntu-base-20.04.3-base-arm64.tar.gz -C test/
		sudo cp -a /usr/bin/qemu-aarch64-static test/usr/bin/

		#option 2
		cd test
		sudo qemu-debootstrap --arch=arm64 --variant=minbase focal ./ http://mirrors.ustc.edu.cn/ubuntu-ports/

		sudo cp /etc/apt/sources.list ./etc/apt/sources.list
		cd ..

		sudo chroot test/

		#change root passwd
		passwd
		
		echo "nameserver xxxx" >> /etc/resolv.conf 
		apt update

		#install gui package
		apt install xorg -y
		apt install --no-install-recommends lightdm-gtk-greeter lightdm openbox mesa-utils vulkan-tools -y
		
		exit
		umount ./test

# QEMU虚拟机启动脚本

为了方便和host共享，启用了9P共享文件系统，QEMU启动脚本：

	qemu-system-aarch64 -machine virt,kernel_irqchip=on,gic-version=3 \
	-cpu host -enable-kvm -smp 4 \
	-m 8G \
	-kernel ./Image \
	-drive file=./rootfs.img,format=raw,if=none,id=hd0,readonly=off \
	-device virtio-blk-device,drive=hd0 \
	-netdev user,id=user0 \
	-device virtio-net-pic,netdev=user0 \
	-device virtio-gpu-pci,virgl=on -display egl-headless \
	-vnc 0.0.0.0:53 \
	-device usb-ehci -device usb-kbd -device usb-tablet -usb \
	-fsdev local,security_model=passthrough,id=fsdev0,path=~/ \
	-device virtio-9p-pci,id=fs0,fsdev=fsdev0,mount_tag=hostshare \
	-append "root=/dev/vda rw" \
	-serial stdio

进入到guest之后，

	mkdir /tmp/host_files
	mount -t 9p -o trans=virtio,version=9p2000.L hostshare /tmp/host_files

## 安装piglit测试OpenGL和Vulkan

在上面生成小系统过程中，

	apt install piglit

进入到guest之后，测试OpenGL：

	cd /usr/lib/aarch64-linux-gnu/piglit/tests/
	piglit run quick ~/piglit-results/quick.out

测试Vulkan：

	cd /usr/lib/aarch64-linux-gnu/piglit/tests/
	piglit run vulkan ~/piglit-results/vulkan.out

# 参考:

[How do you run Ubuntu Server with a GUI?](https://askubuntu.com/questions/53822/how-do-you-run-ubuntu-server-with-a-gui)
[Building Ubuntu Root Filesystem](https://wiki.t-firefly.com/en/ROC-RK3328-PC/linux_build_rootfilesystem.html)
[AM5728-移植ARM Ubuntu 20.04根文件系统](https://blog.csdn.net/weixin_40407893/article/details/118019142)
[Introduction to qemu-debootstrap](https://logan.tw/posts/2017/01/21/introduction-to-qemu-debootstrap/)
[为n1制作aarcm64/arm64 ubuntu rootfs系统](https://www.haiyun.me/page/24/)
[Example Sharing Host files with the Guest](https://www.linux-kvm.org/page/9p_virtio)
[QEMU](https://elinux.org/QEMU)

