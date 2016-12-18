---
title: Git学习笔记（二）
date: 2016-02-19 16:06
categories: 工具
tags: Git
---
### 常用命令：
> 1、初始

	git <verb> --help			查看命令用法(方法之一)
	git init					初始化git仓库(在当前目录下会创建.git目录)

> 2、 远程操作

	git clone URL				从远程clone git仓库
	git remote -v				查看远程仓库使用的 Git 保存的简写与其对应的 URL	
	git pull URL				将数据拉取到你的本地仓库,并合并远程分支到当前分支
	git fetch URL				将数据拉取到你的本地仓库,并不会自动合并或修改你当前的工作
	git push URL				将本地git仓库推送到远程仓库(一次就记住URL了,之后不需要写URL)

> 3、添加文件
	
	git add filename			添加某个文件(该文件处于已跟踪(A)：tracked)
	git add --all , -A			添加所有文件
	
> 4、提交文件

	git commit -m "提交信息"	 	提交当下工作区的文件
	git commit -a -m "提交信息"	 	前两个命令的合并，直接跳过暂存区提交
	git commit						启动文本编辑器写提交信息
	
> 5、查看文件状态

	git status					查看当前文件状态
	git status -s				查看当前文件详细状态列表	
	
> 6、查看日志

	git log						查看提交的日志
	git log	--oneline			查看提交的日志简略信息
	git log	-p -2				查看最近两次提交的日志

> 7、查看修改

	git diff					查看未暂存文件(未add)的修改内容
	git diff --staged, --cached	查看已暂存文件(已add)的修改内容
	
> 8、删除文件

	git rm filename				移除文件
	git rm --cached filename	移除文件,但该文件还留在工作区，但之后不会被跟踪(untracked)
	git rm \*.txt				移除以 .txt 结尾的文件
	
> 9、更改文件名

	git mv oldName newName		更改文件名

> 10、标签操作
	
	git tag -a v1.0 -m "info"	打标签 v1.0:版本号，info:版本信息
	git tag						查看已有标签
	
> 11、分支操作

	git branch branchName		创建分支
	git branch -d branchName	删除分支
	git checkout branchName		切换分支
	git merge branchName		合并分支到当前分支
	git mergetool				启动图形化工具解决冲突
	git branch					查看所有分支
	git branch -v, -vv			查看所有分支的最后一次提交

	
### 补充：
* **rebase**: **变基**，另一种不同于merge的"整合"方式 (命令示例：git rebase master)
* 变基是将一系列提交按照原有次序依次应用到另一分支上，而合并是把最终结果合在一起。
* 请注意：
			
	> 无论是通过变基，还是通过三方合并，整合的最终结果所指向的快照始终是一样的，只不过提交历史不同罢了。
* 准则：

 * 不要对在你的仓库外有副本的分支执行变基
 * 只对尚未推送或分享给别人的本地修改执行变基操作清理历史，从不对已推送至别处的提交执行变基操作