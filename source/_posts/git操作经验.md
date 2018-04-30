title: git操作经验
author: Joy Xu
date: 2018-04-30 10:12:07
tags: git
---

有些git命令实在太长，且不太容易google到，就特地记录下放在这里吧。

# 取远程仓库最新的tag

		# Get new tags from the remote
		$ git fetch --tags

		# Get the latest tag name
		$ latestTag=$(git describe --tags `git rev-list --tags --max-count=1`)

		# Checkout the latest tag
		$ git checkout $latestTag

# 批量更改git commit历史，比如增加前缀后缀之类的

* 加前缀
	`git filter-branch --msg-filter 'echo "bug ###### - \c" && cat' master..HEAD`

* 加后缀
	`git filter-branch -f --msg-filter 'cat && echo "[Reviewer Walsh]"' master..HEAD`

# 使用git回复没有subscribe过的社区邮件

比如我们想回复这个[patch](https://patchwork.kernel.org/patch/10371127/),
拷贝下这行`Message ID	<1525079264-25533-13-git-send-email-eric.auger@redhat.com>`

		｀git send-email --smtp-server smtp.bobberson.com --smtp-user bob --to torvalds@linux-foundation.org --cc linux-kernel@vger.kernel.org  --in-reply-to CA+55aFzNDTm_O8rGYdNw1S99P06u6EeSdttXBvtURJocQT2O0g@mail.gmail.com --annotate <path_to_msg1>｀

参考[Replying to a thread on lkml via git send-email](http://studioidefix.com/2014/06/17/replying-to-lkml/)

# git working flow对比

[Gitflow Workflow](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow)
