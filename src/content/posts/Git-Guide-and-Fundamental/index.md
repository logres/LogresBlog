---
title: Draft Example
published: 2022-09-01
tags: [Markdown, Blogging, Demo]
category: Examples
draft: true
---

凡是对编程有所涉猎的人，想必都对Git有所了解，毕竟在个人开发的角度来说，Git可算是版本控制一把梭；而哪怕是不从事开发的人员，也有不少为GitHub上的开源项目所吸引，在教程的指导下稀里糊涂装上Git，clone项目的。用Git的人或许很多，但是，懂Git的又有几人呢？想用Git，理解Git，还得从Git的原理说开去。

<!--more-->

## What is Git?

Git 是一个开源的分布式版本控制系统。

**开源**：开放源代码
**分布式**：去中心化

## How does it work?

Git采用**键值-数据库**的模式来构建版本控制系统。

### 基础结构

Git系统由三个部分组成：

1. 工作区：源文件存储区，用于工作
2. 版本库：存储历史版本
3. 暂存区：暂时存放一次版本资料的空间，Git区别于其余版本控制系统的设计组成部分

### 思想

Git在版本库中以**blob**的形式存放被提交的文件。

在生成blob文件时，Git对文件进行压缩，同时采用SHA-1算法，得到20字节的hash值，以40个16进制数字表示，前两数字充当文件夹名（此处为版本库中的文件夹），后38个数字则为文件名。
注意到20字节的hash值冲突概率极低，故不考虑碰撞现象。

这就解决了文件存放的问题。

Git以**tree**的形式解决文件夹（工作区中的）的表示形式及组织文件的问题。

tree中包含其统筹下文件及文件夹的索引（指针）以及文件名、文件权限信息。

版本库是由众多文件构成的，而版本库中的所有对象又都是由hash值来区别的。直接在版本库里生成tree再进行添加、修改文件，将改变hash值，不符合设计理念。

解决方法：开辟**暂存区**，用于缓存一次版本库的内容，待确认提交后，再一次性生成tree。

Git以**commit**的形式来记录一次版本库提交。

commit中包含当此提交的tree的索引（指针）、作者、提交者、其前一个版本提交commit的索引（指针）以及提交说明信息。

Git以**tag**的形式来为某次提交补充信息。

tag中包含补充信息以及对象指针，采用命名而非哈希值来区分版本提交，较为便利。

### 具体流程

git add 指令，将文件提交至版本库，生成blob，并在暂存区生成索引。

git commit 指令，以前一个版本的commit，整合暂存区的索引，生成新的commit，并保留parent指针，指向前一commit。

## How about branch?

分支管理是版本控制的重要功能。

Git中采用ref即分支引用来区分分支。

master、dev等分支，分别保留了其指导的分支的最新commit的索引。

而Head则保有当下分支的索引，可间接获得分支最新commit。

### 常用操作

git reset --[hard|mixed|soft] [Head]

将版本回滚到[Head]所指示的版本（Head^表示上一个、Head^^上上个、Head~n上n个）
[hard……]则控制工作区的变化：

+ hard 工作区、暂存区、版本库同步变化
+ soft 仅版本库变化

git merge <branch>

将目标分支合并到目前分支

git remote add origin url

关联远程仓库

git push origin master

将本地master分支推送到远程分支

git pull

拉取分支
