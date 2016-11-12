---
title: GIT常用命令
date: 2016-01-14 11:05:17
tags: GIT
---

## HEAD 
	表示当前版本,每次提交HEAD都会指向最新提交记录 , HEAD^ 上一版本, HEAD^^ 上两个版本 , 
	HEAD~n 上N个版本


## git status

	stage file //列出哪些文件在暂存区中
	modify file //哪些文件被修改
	untracked file //哪些文件从未被add过,没有被版本控制跟踪过 

## git log   --pretty=oneline

	提交记录

## git relog

	命令记录,用于重新回到最新的提交, 可以查看提交记录id

## 提交代码

	git init 初始化一个git仓库

	git add filename   //将新加的文件或修改过的文件添加到暂存区

	git commmit -m "备注信息"

	git pull , 提交后再次拉取,如果这时候远端分支有其他人提交了,这里会merge,如果没有冲突则会自动合并,并产生一个commit, 
	如有冲突则需要手动修改冲突,然后add conflict file, 然后再commit, 再pull, 直到没有冲突

	git push , pull之后如果没有冲突,则可以直接push, 这是本地仓库和远端仓库数据一致


## 分支相关

	git branch //查看所有分支, console窗口中会用星号标明当前分支

	git branch branchname //新建分支, 如果分支存在则报错

	git checkout //切换分支

	git checkout -b dev origin/dev //第一次开发时需要从远程拉取分支到本地

	git merge branchname //合并分支到当前分支, 可能有冲突需要和pull差不多的方式解决, 因为pull会merge

	git branch --set-upstream dev origin/dev //设置本地分支和远程分支关联,才可以push 和 pull

## 撤销修改

	git checkout -- filename //使用仓库中的版本覆盖自己的自改,如果已经添加到暂存区(add), 则返回到暂存区的修改状态

	git reset HEAD filename //丢弃暂存区的更改,然后再使用git checkout -- filename覆盖本地修改



## github使用

	在github上新建Repository,默认没有仓库,可以在本地创建仓库并push到github
	echo "# gittest" >> README.md
	git init
	git add README.md
	git commit -m "first commit"
	git remote add origin https://github.com/qifanyang/gittest.git
	git push -u origin master -- 关联本地和远端, 使用-u
	
	还可以push已经存在的仓库,切换到仓库中执行以下命令
	git remote add origin https://github.com/qifanyang/gittest.git
	git push -u origin master

	github默认远程库约定叫origin

## 删除版本库中的big file
	github仓库大小有限制,当空间不足时需要删除仓库中不重要的大文件,但是github仓库中还保留这这个大文件,如何移除

	1.查看空间大小
		git gc //整理
		git count-objects //输出的size-pack 为仓库大小, K为单位


use `java java`


Here is an example of AppleScript:

    tell application "Foo"
        beep
    end tell



>dsfdf 
>sdfdsf 
>sdf


行内样式连接,少了个冒号

This is [an example](http://example.com/ "Title") inline link.


[id](<http://example.com/>  "Optional Title Here")


引用的方式

[node.js]

[node.js]: <http://nodejs.org> "官网"

```js

public 

```