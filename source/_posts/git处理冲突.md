---
title: git处理冲突
date: 2019-03-05 22:51:59
tags:
---
1.停止rebase,rebase 是一个状态
  git rebase --abort

2.git fetch
  git rebase origin/master
  远端覆盖本地分支（reset回退版本）
  $ git reset origin/master --hard
  推送远端
  $ git push origin HEAD:refs/for/master

3.//冲突
  $ git reset origin/master --hard
  $ git fetch
  $ git rebase origin master
  处理冲突
  $ git staus
  $ git rebase --continue
  $ git push origin HEAD:refs/for/master

4.//处理abandon
  修改文件
  git add .
  git commit --amend
  git push origin HEAD:refs/for/master

5.远程分支回滚的三种方法：
  自己的分支回滚直接用reset
  公共分支回滚用revert
  错的太远了直接将代码全部删掉，用正确代码替代

其他操作：
git pull --rebase origin master
git rebase --continue
git push origin HEAD:refs/for/master
git checkout -b name
git branch -a
git remote add ...
git remote -v
git checkout -b <本地分支名> <远程主机名>/<远程分支名>
git checkout filename  恢复工作区
git reset filename     恢复暂存区
