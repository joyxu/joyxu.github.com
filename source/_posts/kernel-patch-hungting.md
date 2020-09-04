
---
title: Linux patch狩猎
author: "Joy Xu"
date: 2020-09-04 19:44:44
tags: [Linxu Kernel]
---

最近要冲linux的补丁数量了，写个总结，有哪些方法可以快速找到补丁：

# Coccinelle静态检查

利用Coccinelle通过静态检查的方式，根据输出的提示，找bug, 从官网下载好之后，
按照install说明进行编译安装后，直接进入到kernel目录，运行下面命令，之后根据提示
找补丁：
	sudo apt-get install -y coccinelle
	make coccicheck MODE=report ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-

运行效果如下图：
	[coccicheck效果图](.images/coccinelle.PNG)

# 使用内核自带的检查

直接运行 make W=1 C=2

# 在线Coverity检查 - 难度较大

访问https://scan.coverity.com/

# google的用户态猴子测试 - 难度较大

访问https://syzkaller.appspot.com

# 参考

* [Coccinelle介绍](https://kernel-recipes.org/en/2013/automating-source-code-evolutions-using-coccinelle/)
* [patch狩猎专家](https://www.slideshare.net/ennael/kernel-recipes-2019-hunting-and-fixing-bugs-all-over-the-linux-kernel-178217304)
* [syzkaller](https://xz.aliyun.com/t/5079)
