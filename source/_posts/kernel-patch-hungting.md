
---
title: Linux patch狩猎
author: "Joy Xu"
date: 2020-09-04 19:44:44
tags: [Linux Kernel]
---

最近要冲linux的补丁数量了，写个总结，有哪些方法可以快速找到补丁。
找到想改的补丁后，先在对应的[patchwork邮件列表](https://lore.kernel.org/patchwork/project/lkml/list/)中搜下，看看这个patch是否有其他人已经发出来了，避免做无用功。
补丁做好之后，还是按照老步骤保证补丁质量：
	
	make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- O=../kernel-dev.build -j32

	git format-patch -1
	./scripts/checkpatch.pl *.patch

	git send-email \
	--dry-run --annotate \
	--cc-cmd="./scripts/get_maintainer.pl --norolestats" \
	--to=xxx@xxx.com \
	--cc=self@xxx.com \
	xxx.patch

一般fix的patch ，还需要把下面的输出添加到patch的commit消息里面去：

	git log --pretty=oneline --abbrev-commit xxx.c

添加的消息格式稍微修改下，再添加到git commit消息中，比如：

	Fixes: 54a4f0239f2e ("KVM: MMU: make kvm_mmu_zap_page() return the number of pages it actually freed")

详细步骤可以参考：https://www.kernel.org/doc/html/v5.9-rc5/process/submitting-patches.html

# Coccinelle静态检查

利用Coccinelle通过静态检查的方式，根据输出的提示，找bug, 从官网下载好之后，
按照install说明进行编译安装后，直接进入到kernel目录，运行下面命令，之后根据提示
找补丁：

	sudo apt-get install -y coccinelle
	make coccicheck MODE=report ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-

运行效果如下图：
	![coccicheck效果图](/images/coccinelle.PNG)

# 使用内核自带的检查

直接运行 make W=1 C=2

# 在线Coverity检查 - 难度较大

访问https://scan.coverity.com/

# google的用户态猴子测试 - 难度较大

访问https://syzkaller.appspot.com

# Kernel Self Protection Project

主页http://kernsec.org/wiki/index.php/Kernel_Self_Protection_Project上有bug跟踪
系统介绍，访问https://github.com/KSPP/linux/issues

这个模块现在有专门的maintainer https://git.kernel.org/pub/scm/linux/kernel/git/gustavoars/linux.git/ 

# ClangBuiltLinux静态检查

访问https://github.com/ClangBuiltLinux/linux/issues

# 参考

* [Coccinelle介绍](https://kernel-recipes.org/en/2013/automating-source-code-evolutions-using-coccinelle/)
* [patch狩猎专家](https://www.slideshare.net/ennael/kernel-recipes-2019-hunting-and-fixing-bugs-all-over-the-linux-kernel-178217304)
* [syzkaller](https://xz.aliyun.com/t/5079)
