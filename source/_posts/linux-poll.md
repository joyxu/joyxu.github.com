
title: linux poll机制
author: Joy Xu
date: 2020-09-23 20:34:06
tags: linux kernel, poll
---

# IO机制

IO机制一般分为4种：阻塞，非阻塞，同步和异步。
这篇[文章](https://www.cnblogs.com/lovingprince/archive/2011/05/17/2166241.html)
讲的很清楚，这几种IO的区别，借用下别人的图。

![Linux IO区别](/images/linux-io.gif)

# linux poll

这篇[文章](https://www.cnblogs.com/loyenWang/p/12622904.html)基本上讲的很透彻了，
最好的办法还是要自己多上上手，另外整体上可以看看这篇[文章](https://www.oreilly.com/library/view/linux-device-drivers/0596000081/ch05s03.html)，
越来越觉得能把文章写得精简而达意是门高深的技术了。同时借用下别人的图。

![poll table entry and wait queue entry](/images/poll-table.png)

# 总结
最后总结下，驱动里面需要加一个wait queue head，提供poll这个ops。
在poll ops函数中，调用poll_wait把自己加入到poll table的entity里面去，针对情况
返回pollin等，如果返回0，则用户态调用poll的时候则一直阻塞。

# 参考

* [Linux kernel concurrency cheat sheet](https://blogs.oracle.com/linux/post/linux-kernel-concurrency-cheat-sheet)
