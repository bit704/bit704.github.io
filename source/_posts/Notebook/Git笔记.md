---
title: Git笔记
cover: https://tva4.sinaimg.cn/large/0077Un8Egy1h5et13d2goj30sg0kbmyt.jpg
banner: https://tva4.sinaimg.cn/large/0077Un8Egy1h5et13d2goj30sg0kbmyt.jpg
layout: post
categories: [Notebook]
tags: [Git]
mathjax: false
---

一个开源的分布式版本控制系统

<!-- more -->

## 一.基础概念

### 1.工作区域

![区域构成1](https://tva3.sinaimg.cn/large/0077Un8Egy1h5fds7sscvj30wk09gq5i.jpg)

**workspace**: 工作区，电脑上当前可见的目录。

**Index/Stage**: 暂存区，临时存放变动。实际上是一个文件，保存即将提交到文件列表的信息。位于.git/index。

**Repository**: 版本库，即.git。其中HEAD指向最新放入版本库的版本。

**Remote**: 远程仓库，托管代码的服务器。

![区域构成2 ](https://tvax2.sinaimg.cn/large/0077Un8Egy1h5fdsccpllj30iz09xmxu.jpg)

### 2.文件状态

![文件状态](https://tva3.sinaimg.cn/large/0077Un8Egy1h5fdsgl5vzj30m8096dhj.jpg)

GIT不关心文件两个版本之间的具体差别，而是关心文件的整体是否有改变，若文件被改变，在添加提交时就生成文件新版本的快照，而判断文件整体是否改变的方法就是用**SHA-1算法**计算文件的校验和。

**Untracked**: 文件未跟踪，位于工作区。没有加入到暂存区，不参与版本控制。通过git add 变为Staged。

**Unmodify**: 文件位于版本库，未修改。即版本库中的文件快照内容与工作区中完全一致。 这种文件有两个去处，如果它被修改，变为Modified；如果使用git rm移出版本库，则成为Untracked文件。

**Modified**: 文件已修改，位于工作区。这种文件有两个去处，通过git add可进入暂存staged状态；使用git checkout 则丢弃修改，返回unmodify或staged。

**Staged**: 文件位于暂存区。执行git commit则将修改同步到版本中。执行git reset HEAD filename取消暂存,文件状态为Modified。

![文件状态转变](https://tva3.sinaimg.cn/large/0077Un8Egy1h5fdsp0nvdj30t30k8drf.jpg)

## 二.命令

### 1.基础

```bash
git init <dir>    #新建git仓库,如果带dirname，则新建目录
git clone --recursive <url>   #克隆远程仓库，带--recursive则克隆子仓库

git status <file>  #查看文件状态，不带<file>默认查看所有文件

git add <file>      # 将文件从工作区添加到暂存区
git restore <file>  # 弃置工作区的变动

git rm <file>  #将文件从工作区、暂存区、版本库上均删除

git rm --cached <file>  # 将文件从暂存区和版本库上删除，但是本地工作区还有。再次add之后会重新添加，使用.gitignore可以避免。
git restore --staged <file>  # 将文件的更改从暂存区撤销

git checkout .
git checkout -- <file>  #撤销在工作区做的修改，使用暂存区或版本库的文件替换工作区的文件。如果自修改后还没有被放到暂存区，撤销修改就回到和版本库一模一样的状态。如果已经添加到暂存区后，又作了修改，撤销修改就回到添加到暂存区后的状态。总之，就是让这个文件回到最近一次git commit或git add时的状态。
git checkout HEAD .
git checkout HEAD <file>  #使用HEAD 指向的 master 分支中的全部或者部分文件替换暂存区和工作区中的文件

git commit -m "备注"    # 一次提交暂存区的所有更改到版本库，并备注

git reset HEAD <file>  #使用HEAD 指向的 master 分支替换暂存区，工作区不受影响

# 同步submodule
git submodule sync --recursive
git submodule update --init --recursive
```

### 2.清空main分支的所有提交记录

```bash
git checkout --orphan lastest_branch  # 切换到全新的分支
git add .                     # 缓存所有文件
git commit -m "first commit"  # 提交
git branch -d main          # 删除main分支
git branch -m main          # 重命名当前分支为main
git push -f origin main      # 强行推送到远程的main分支
```



## 参考

[Git 工作区、暂存区和版本库  菜鸟教程 (runoob.com)](https://www.runoob.com/git/git-workspace-index-repo.html)

[【Git】---工作区、暂存区、版本库、远程仓库 - 雨点的名字 - 博客园 (cnblogs.com)](https://www.cnblogs.com/qdhxhz/p/9757390.html)

[Git 少用 Pull 多用 Fetch 和 Merge - 技术翻译 - OSCHINA 社区](https://www.oschina.net/translate/git-fetch-and-merge?print=)

[git submodule添加、更新和删除 - JYRoy - 博客园 (cnblogs.com)](https://www.cnblogs.com/jyroy/p/14367776.html)

[git commit --amend - 简书 (jianshu.com)](https://www.jianshu.com/p/7d40838883af)

[git commit回退三种姿势_DiuDiu_yang的博客-CSDN博客_git 回退commit](https://blog.csdn.net/qq_41261490/article/details/108119801)