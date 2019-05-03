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
apt-get install resolveconf && echo 'nameserver 8.8.8.8' > /etc/resolvconf/resolv.conf.d/tail
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
		sudo apt install -y qemu-efi libvirt-clients libvirt-daemon-system bridge-utils virt-manager

## host kernel配置

		CONFIG_NETFILTER_XT_NAT=y
		CONFIG_NF_NAT_MASQUERADE_IPV4=y
		CONFIG_IP_NF_NAT=y
		CONFIG_IP_NF_TARGET_MASQUERADE=y
		CONFIG_IP_NF_TARGET_NETMAP=y
		CONFIG_IP_NF_TARGET_REDIRECT=y
		CONFIG_NF_NAT_MASQUERADE_IPV6=y


## avocado和avocado-vt安装

建议使用avocado的lts版本。

		sudo apt-get install -y python git gcc python-dev libvirt-dev \
        	libffi-dev libssl-dev libyaml-dev xz-utils \
	        liblzma-dev make python-libvirt arping wget numactl zlib1g-dev python3-pip libosinfo-bin

		sudo update-alternatives --install /usr/bin/python python /usr/bin/python3 10

		git clone https://github.com/avocado-framework/avocado.git
		cd avocado
		git fetch -t 62.0 && git reset --hard 62.0
		make develop
		python setup.py install

		avocado run /bin/true

		git clone https://github.com/avocado-framework/avocado-vt.git
		务必使用最新的，有些patch19年1月份才刚刚合入
		cd avocado-vt
		make requirements
		python setup.py install

		cd ../avocado
		make link

		avocado plugins
		avocado vt-bootstrap --vt-type libvirt --vt-guest-os Fedora.27.aarch64 --yes-to-all
		avocado list --vt-type libvirt --vt-guest-os Fedora.27.aarch64.arm64-pci  --verbose

安装完成之后，确认avocado和avocado-vt均没有问题。

## avocado调用libvirt

## avocado测试用例配置

		export AEXPECT_DEBUG=true

		apt-get install resolvconf
		echo "nameserver 8.8.8.8" > /etc/resolvconf/resolv.conf.d/tail
		resolvconf -u


## libvirt测试用例

	wget https://dl.fedoraproject.org/pub/fedora-secondary/releases/27/Server/aarch64/iso/
	/var/lib/avocado/data/avocado-vt/isos/linux/Fedora-Server-dvd-aarch64-27-1.6.iso
	avocado run io-github-autotest-qemu.unattended_install.cdrom.extra_cdrom_ks.default_install.aio_native --vt-type libvirt --vt-guest-os Fedora.27.aarch64.arm64-pci 


	undefine avocado-vt-vm1 --nvram

	avocado run --vt-type libvirt --vt-guest-os Fedora.27.aarch64.arm64-pci   import.default_install.aio_native


	grep -v -e '^#' -e '^$' /var/lib/avocado/data/avocado-vt/backends/libvirt/cfg/default_tests | xargs avocado run --vt-type libvirt --vt-guest-os Fedora.27.aarch64.arm64-pci


	total: 122
	failed:21
	   type_specific.io-github-autotest-libvirt.virsh.snapshot.live.no_halt ; no fix
	   type_specific.io-github-autotest-libvirt.virsh.snapshot.live.halt ;no fix

	   type_specific.io-github-autotest-libvirt.virsh.nodeinfo.no_option ; fixed

	   type_specific.io-github-autotest-libvirt.virsh.save.normal_test.acl_test.paused_option.no_progress ;
	   type_specific.io-github-autotest-libvirt.virsh.save.normal_test.acl_test.paused_option.show_progress ;no fix

	   type_specific.io-github-autotest-libvirt.virsh.managedsave.status_error_no.name_option.paused_status.no_opt.no_progress;
	   type_specific.io-github-autotest-libvirt.virsh.managedsave.status_error_no.name_option.paused_status.no_opt.show_progress
	   type_specific.io-github-autotest-libvirt.virsh.managedsave.status_error_no.name_option.paused_status.paused_opt.no_progress
	   type_specific.io-github-autotest-libvirt.virsh.managedsave.status_error_no.name_option.paused_status.paused_opt.show_progress
	   type_specific.io-github-autotest-libvirt.virsh.managedsave.status_error_no.name_option.paused_status.running_opt.no_progress
	   type_specific.io-github-autotest-libvirt.virsh.managedsave.status_error_no.name_option.paused_status.running_opt.show_progress

	   type_specific.io-github-autotest-libvirt.virsh.managedsave.status_error_no.name_option.normal_status.no_opt.no_progress
	   type_specific.io-github-autotest-libvirt.virsh.managedsave.status_error_no.name_option.normal_status.no_opt.show_progress
	   type_specific.io-github-autotest-libvirt.virsh.managedsave.status_error_no.name_option.normal_status.paused_opt.no_progress
	   type_specific.io-github-autotest-libvirt.virsh.managedsave.status_error_no.name_option.normal_status.paused_opt.show_progress
	   type_specific.io-github-autotest-libvirt.virsh.managedsave.status_error_no.name_option.normal_status.running_opt.no_progress
	   type_specific.io-github-autotest-libvirt.virsh.managedsave.status_error_no.name_option.normal_status.running_opt.show_progress

	   type_specific.io-github-autotest-libvirt.virsh.dump.negative_test.no_dump_file;

	   type_specific.io-github-autotest-libvirt.virsh.dump.positive_test.non_acl.bypass_cache_dump ;fixed
	   type_specific.io-github-autotest-libvirt.virsh.dump.positive_test.acl_test.bypass_cache_dump ;

	   type_specific.io-github-autotest-libvirt.virsh.destroy.normal_test.acl_test.paused_option

	   type_specific.io-github-autotest-libvirt.virsh.console.normal_test.non_acl.valid_domid
	   type_specific.io-github-autotest-libvirt.virsh.console.normal_test.acl_test.valid_domid


	   cancel:
	   type_specific.io-github-autotest-libvirt.virsh.domjobabort.normal_test.migrate_option.running_option.uuid_option



	mkdir -p /usr/share/qemu-kvm
	cp source/qemu/pc-bios/*virtio.rom /usr/share/qemu-kvm
	--for below test cases --
	type_specific.io-github-autotest-libvirt.virtual_network.iface_update.positive_test.update_link_with_rom

	iface_hotplug

	virtual_disks.usb
