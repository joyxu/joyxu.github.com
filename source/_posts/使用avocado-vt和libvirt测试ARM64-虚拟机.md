title: 使用avocado-vt和libvirt测试ARM64 虚拟机
author: Joy Xu
date: 2019-02-06 10:47:00
tags: [Virtualization, Linux Kernel]
---

本文记录下在ARM64平台上使能avocado-vt和libvirt的过程，
特别是趟过的坑。面向的读者是自己，可能会比较跳跃，不连贯。

# 准备工作

## host操作系统准备

1. 由于只有libvirt 4.0才支持arm平台上的pcie hotplug，如果不想自己编译的话，需下载ubuntu 18.04之后的版本。
这里提供一个下载链接：[Ubuntu 18.04 arm64 cloud](https://cloud-images.ubuntu.com/bionic/current/bionic-server-cloudimg-arm64-root.tar.xz)
2. 解压上述镜像后，修改`/etc/passwd`把root那一行的x修改成空格，这样root账号就没有密码了,串口可以直接用root登录。
3. `touch /etc/cloud/cloud-init.disabled` 来禁用cloud-init服务。
4. 由于cloud-init的原因，需要运行`dpkg-reconfigure openssh-server`允许ssh登录
5. 修改/etc/resolve.conf 添加`nameserver 8.8.8.8`
6. 修改/etc/apt/source.list, un-comment deb-src xxx main 那一行，支持build-dep。

# 使能步骤

## QEMU编译

		sudo apt-get build-dep qemu
		git clone git://git.qemu.org/qemu.git qemu
		cd qemu
		git fetch -t v3.0.0 && git reset --hard v3.0.0
		./configure --target-list=aarch64-softmmu
		make && make install

如果想直接安装依赖包的话，也可以执行以下命令:

		apt-get install -y build-essential zlib1g-dev pkg-config libglib2.0-dev binutils-dev  autoconf libtool libssl-dev libpixman-1-dev libpython-dev python-pip  qemu-efi bridge-utils 

安装完以后，请运行以下命令确认系统中的版本:

		qemu-system-aarch64 --version

## libvirt安装

执行以下命令:
		sudo apt install -y qemu-efi libvirt-clients libvirt-daemon-system bridge-utils virt-manager  -y

brctl addbr br0

## avocado和avocado-vt安装

建议使用avocado，62.0版本，后面的版本尝试过程中总有些莫名其妙的问题。

		sudo apt-get install -y python git gcc python-dev libvirt-dev \
        	libffi-dev libssl-dev libyaml-dev xz-utils \
	        liblzma-dev make python-libvirt arping wget numactl zlib1g-dev

		git clone https://github.com/avocado-framework/avocado.git
		cd avocado
		git fetch -t 62.0 && git reset --hard 62.0
		make requirements
		python setup.py install

		git clone https://github.com/avocado-framework/avocado-vt.git
		cd avocado-vt
		git fetch -t 62.0 && git reset --hard 62.0
		make requirements
		python setup.py install

		cd ../avocado
		make link

安装完成之后，确认avocado和avocado-vt均没有问题。

## avocado调用libvirt

## avocado测试用例配置

