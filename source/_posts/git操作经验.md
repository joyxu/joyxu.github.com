title: git操作经验
author: Joy Xu
date: 2018-04-30 10:12:07
tags: git
---

有些git命令实在太长，且不太容易google到，就特地记录下放在这里吧。

# 调试打印

git命令之前加`GIT_TRACE=2`或者`GIT_CURL_VERBOSE=1`

# 查看远程更新

		git remote update
		git status

# 查看merge commmit历史

		git log --oneline --graph

# 发送带cover letter的series

		sendemail.suppresscc=all //一定要在git config加上，避免误发
		git send-email -7 --cover-letter --annotate --subject-prefix="PATCH v2" --to xxx@xxx.com

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

* 中间插入
	`git filter-branch -f --msg-filter 'sed "2 s/^/\n xxx \n"' HEAD~1..HEAD`  

* 修改作者信息
	`git filter-branch -f --env-filter "GIT_AUTHOR_NAME='xxx' GIT_AUTHOR_EMAIL='xxx' GIT_COMMITTER_NAME='xxx' GIT_COMMITTER_EMAIL='xxx'" HEAD~1..HEAD`  

* 插入sign off
	`git filter-branch -f --msg-filter 'git interpret-trailers --where start --trailer "Signed-off-by: xxx <xxx@xxx.com>"' HEAD~1..HEAD`  

# 使用git回复没有subscribe过的社区邮件

比如我们想回复这个[patch](https://patchwork.kernel.org/patch/10371127/),
拷贝下这行`Message ID	<1525079264-25533-13-git-send-email-eric.auger@redhat.com>`

		｀git send-email --smtp-server smtp.bobberson.com --smtp-user bob --to torvalds@linux-foundation.org --cc linux-kernel@vger.kernel.org  --in-reply-to CA+55aFzNDTm_O8rGYdNw1S99P06u6EeSdttXBvtURJocQT2O0g@mail.gmail.com --annotate <path_to_msg1>｀

参考[Replying to a thread on lkml via git send-email](http://studioidefix.com/2014/06/17/replying-to-lkml/)

# git working flow对比

* [A successful Git branching model](http://nvie.com/files/Git-branching-model.pdf)
* [Gitflow Workflow](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow)

# git 拆分子目录成为一个独立git仓库

* 保留目标子目录的git commit history:  `git filter-branch --prune-empty --subdirectory-filter FOLDER-NAME  BRANCH-NAME`
* 添加目标git repo: `git remote add xxx`
* 提交到目标git repo: `git push xxx branch-name:master`


# git 拆分patch

假设当前有几个改动需要重新拆分，简要的命令如下：

		git reset --soft //reset到这几个改动之前的commit点
		git checkout HEAD //按照提示，把这些改动恢复到unstaged
		/*
		 按照提示，选Y则把这部分改动合入，选N则放弃，选e则手动合入；
		 手动情况下：
			* 不想要的新增的，直接删掉
			* 不想要的删除，把行首的-号去掉
			* 想要的改动，保留行首符号
		 */
		git add -p
		git commit -s -u //针对这部分，加上commit 消息
		git rebase -i HEAD~2 //修改最近两次改动
