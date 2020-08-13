---
layout: post
title: Git 利用sparse-checkout 只下载子文件夹
category: 默认分类
tags: [git]
keywords: git
---

#Git 利用sparse-checkout 只下载子文件夹/n
Windows git 利用sparse-checkout 只下载子文件夹
=====================================

2017年11月26日 09:58:31 [doujiang_zheng](https://me.csdn.net/doujiang_zheng) 阅读数：2000

因为全部项目文件太大，而且只对某几个子文件夹感兴趣，印象中git有个sparse-checkout的功能，因此实践一番，遇到了一些问题，以[Tensorflow Models](https://github.com/tensorflow/models) 为例分享一下。

[Segmentfault的分享](https://segmentfault.com/q/1010000006666948)

按照`爱睡觉的小猫咪` 答案所述

    git init models && cd models
    git config core.sparsecheckout true //设置允许克隆子目录
    echo official/resnet/* >> .git/info/sparse-checkout //设置要克隆的仓库的子目录路径
    git remote add origin https://github.com/tensorflow/models
    git pull origin master 

这种方法的`git pull origin master`存疑，我遇到过`error: Sparse checkout leaves no entry on the working directory`的错误，按照下一方法的`git checkout master`则没有出现错误。

[StackOverflow的分享](https://stackoverflow.com/questions/23289006/on-windows-git-error-sparse-checkout-leaves-no-entry-on-the-working-directory)

    git clone -n https://github.com/tensorflow/models
    cd tensorflow
    git config core.sparsecheckout true
    echo official/resnet >> .git/info/sparse-checkout
    git checkout master

值得注意的是，无论哪种方法都需要下载`.git`文件夹，这是git的元数据存储。 大项目的`.git`元数据通常也比较庞大，比如`Tensorflow` 元数据133.31MB，`Models` 的元数据243.66MB，下载依然要不少时间。  
当想追加子文件夹时，继续`echo official/mnist/* >> .git/info/sparse-checkout` 再`git checkout master` 即可。

