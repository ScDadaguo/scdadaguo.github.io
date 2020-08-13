---
layout: post
title: git rebase详解
category: 默认分类
tags: [git]
keywords: git
---

#git rebase详解/n## [【Git系列】git rebase详解]()

转：https://www.cnblogs.com/pinefantasy/articles/6287147.html

git合并代码方式主要有两种方式，分别为：  
1、merge处理，这是大家比较能理解的方式。  
2、rebase处理，中文此处翻译为衍合过程。

git rebase操作讲解例子：

 1. cd /usr/local/test
 2. mkdir hellogit
 3. cd hellogit # 创建hellogit目录
 4. git init # 初始化git项目
 5. vim readme # 新建readme文件，往里边添加内容
 6. git add . # 提交内容
 7. git commit -m 'init project c1' # git系统默认创建一个master分支

# 接着我们创建一个dev分支，在dev分支上添加内容

 1. git checkout -b dev # 此处其实是两步git branch dev加上git checkout dev
 2. vim readme # 在原来基础上增加上内容
 3. git add .
 4. git commit -m 'add hello world c2' \# 切换回到master分支
 5. git checkout master
 6. vim readme # 编辑readme文件，在第二行增加hello world from master内容

# 此处先埋个点，因为此处会和dev分支上做的修改冲突

 1. git add .
 2. git commit -m 'add hello world c3' vim hello.py # 新添加一个hello.py文件
 3. git add .
 4. git commit -m 'add hello.py c4' \# 切换回到dev分支
 5. git checkout dev
 6. vim helloworld.py # 添加上helloworld.py文件
 7. git add .
 8. git commit -m 'add helloworld.py c5'

至此，我们简单分析下情况为：

master分支，节点链表指向为：c1<--c3<--c4  
dev分支，节点链表指向为：c1<--c2<--c5  
master分支和dev分支祖先为c1，假定在master分支上做git merge dev合并，得到的提交历史为：  
c1<--c2<--c3<--c4<--c5<--c6（c1、c4、c5做了一次三方合并发现冲突，手工处理完毕后git add/commit增加了提交节点c6）  
采用git merge dev处理提交log是按照时间戳先后顺序的。

假定采用的是git rebase处理过程为：

git checkout dev
git rebase master # 将dev上的c2、c5在master分支上做一次衍合处理
# git提示出现了代码冲突，此处为之前埋下的冲突点，处理完毕后
git add readme # 添加冲突处理后的文件
git rebase --continue # 加上--continue参数让rebase继续处理

_此处处理后的节点为：_

_c1 c3 c4 c2 c5 # 此处不是按照时间顺序处理的  
综合表现，git rebase可以得到一个更加简洁的提交历史，无需多了c6。  
处理完毕后，git checkout master加上git merge dev，git会智能采用f-f处理。_

# 总结为：  
git rebase过程相比较git merge合并整合得到的结果没有任何区别，但是通过git rebase衍合能产生一个更为整洁的提交历史。  
如果观察一个衍合过的分支的历史提交记录，看起来会更清楚：仿佛所有修改都是在一根线上先后完成的，尽管实际上它们原来是同时并行发生的。

一般我们使用衍合的目的，是想要得到一个能在远程分支上干净应用的补丁，比如某个项目你不是维护者，但是想帮点忙，最好使用衍合处理。  
先在自己的一个分支进行开发，当准备向主项目提交补丁的时候，根据最新的orgin/master进行一次衍合操作然后再提交，这样维护者就不需要任何整合工作。

# 实际为：把解决分支补丁同最新主干代码之间的冲突的责任，划转给由提交补丁的人来解决。  
作为维护项目的人只需要根据你提供的仓库地址做一次快进合并，或者直接采纳你提交的补丁。

衍合的风险，请务必遵循如下准则：  
一旦分支中的提交对象发布到公共仓库，就千万不要对该分支进行衍合操作。

