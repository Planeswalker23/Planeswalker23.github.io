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

## 合并代码
1. 将dev分支上的代码合并到master上
    1. 将dev分支的代码提交并推送至远程仓库，保持dev分支工作树干净
    2. `git checkout master`切换分支到master
    3. `git merge dev`将dev分支的代码合并到当前分支（若有冲突则需处理后才能合并）
2. 将dev分支上的某一次commit合并到master分支上
	1. `git checkout master`切换分支到master
	2. `git cherry-pick 9hu7sy`将id为9hu7sy的这次提交合并到master分支