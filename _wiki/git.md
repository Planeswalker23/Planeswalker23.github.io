---
layout: wiki
title: 常用的Git命令
categories: [git]
description: 常用的Git命令
keywords: git
---

## Tag标签
 - `git tag -a tag_name -m "tag_commit"`创建Git仓库标签，标签名为tag_name，备注为tag_commit
 - `git tag -d tag_name`删除tag_name标签
 - `git push origin --tags`发布标签到远程仓库，若将--tag改成特定标签名，则将制定的标签推送到远程仓库
 
 -----

## 合并代码
1. 将dev分支上的代码合并到master上
    1. 将dev分支的代码提交并推送至远程仓库，保持dev分支工作树干净
    2. `git checkout master`切换分支到master
    3. `git merge dev`将dev分支的代码合并到当前分支（若有冲突则需处理后才能合并）
2. 将dev分支上的某一次commit合并到master分支上
	1. `git checkout master`切换分支到master
	2. `git cherry-pick 9hu7sy`将id为9hu7sy的这次提交合并到master分支

-----

## 代码回退
> 已`commit`但未`push`
1. 查询`commit`历史，拿到`commitId`: `git log`
2. 代码回退到此次提交: `git reset --hard commitId`

-----
## `git pull`报错
执行`git pull`命令，控制台出现如下错误（当前分支名为`dev.2.5.5-bugfix`）：
```
Fetching origin
There is no tracking information for the current branch.
Please specify which branch you want to merge with.
See git-pull(1) for details.

    git pull <remote> <branch>

If you wish to set tracking information for this branch you can do so with:

    git branch --set-upstream-to=origin/<branch> dev.2.5.5-bugfix
```
出现该问题的原因是本地分支和远程分支没有建立联系，可以使用最后一段`git branch --set-upstream-to=origin/<branch>`来解决这个问题。

查询本地分支与远程分支关联关系: `git branch -vv`