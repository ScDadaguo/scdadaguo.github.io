---
layout: post
title:  Docker Volume 记录
category: 默认分类
tags: [docker,volume]
keywords: docker,volume
---

# Docker Volume 记录/n
Docker Volume 记录
============================
转：[Docker学习笔记（6）——Docker Volume](https://www.jianshu.com/p/ef0f24fd0674)

Docker的数据持久化主要有两种方式：

*   bind mount
*   volume

Docker的数据持久化即使数据不随着container的结束而结束，数据存在于host机器上——要么存在于host的某个指定目录中（使用bind mount），要么使用docker自己管理的volume（/var/lib/docker/volumes下）。

### bind mount

bind mount自docker早期便开始为人们使用了，用于将host机器的目录mount到container中。但是bind mount在不同的宿主机系统时不可移植的，比如Windows和Linux的目录结构是不一样的，bind mount所指向的host目录也不能一样。这也是为什么bind mount不能出现在Dockerfile中的原因，因为这样Dockerfile就不可移植了。

将host机器上当前目录下的host-data目录mount到container中的/container-data目录：

    docker run -it -v $(pwd)/host-dava:/container-data alpine sh
    

有几点需要注意：

*   host机器的目录路径必须为全路径(准确的说需要以`/`或`~/`开始的路径)，不然docker会将其当做volume而不是volume处理
*   如果host机器上的目录不存在，docker会自动创建该目录
*   如果container中的目录不存在，docker会自动创建该目录
*   如果container中的目录已经有内容，那么docker会使用host上的目录将其覆盖掉

### 使用volume

volume也是绕过container的文件系统，直接将数据写到host机器上，只是volume是被docker管理的，docker下所有的volume都在host机器上的指定目录下/var/lib/docker/volumes。

将my-volume挂载到container中的/mydata目录：

    docker run -it -v my-volume:/mydata alpine sh
    

然后可以查看到给my-volume的volume：

    docker volume inspect my-volume
    [
        {
            "CreatedAt": "2018-03-28T14:52:49Z",
            "Driver": "local",
            "Labels": null,
            "Mountpoint": "/var/lib/docker/volumes/my-volume/_data",
            "Name": "my-volume",
            "Options": {},
            "Scope": "local"
        }
    ]
    

可以看到，volume在host机器的目录为`/var/lib/docker/volumes/my-volume/_data`。此时，如果my-volume不存在，那么docker会自动创建my-volume，然后再挂载。

也可以不指定host上的volume：

    docker run -it -v /mydata alpine sh
    

此时docker将自动创建一个匿名的volume，并将其挂载到container中的/mydata目录。匿名volume在host机器上的目录路径类似于：`/var/lib/docker/volumes/300c2264cd0acfe862507eedf156eb61c197720f69e7e9a053c87c2182b2e7d8/_data`。

除了让docker帮我们自动创建volume，我们也可以自行创建：

    docker volume create my-volume-2
    

然后将这个已有的my-volume-2挂载到container中:

    docker run -it -v my-volume-2:/mydata alpine sh
    

需要注意的是，与bind mount不同的是，如果volume是空的而container中的目录有内容，那么docker会将container目录中的内容拷贝到volume中，但是如果volume中已经有内容，则会将container中的目录覆盖。请参考[这里](https://medium.com/@yaofei/docker-volume-what-i-learned-27134081d6d9)。

### Dockerfile中的VOLUME

在Dockerfile中，我们也可以使用VOLUME指令来申明contaienr中的某个目录需要映射到某个volume：

    #Dockerfile
    VOLUME /foo
    

这表示，在docker运行时，docker会创建一个匿名的volume，并将此volume绑定到container的/foo目录中，如果container的/foo目录下已经有内容，则会将内容拷贝的volume中。也即，Dockerfile中的`VOLUME /foo`与`docker run -v /foo alpine`的效果一样。

Dockerfile中的VOLUME使每次运行一个新的container时，都会为其自动创建一个匿名的volume，如果需要在不同container之间共享数据，那么我们依然需要通过`docker run -it -v my-volume:/foo`的方式将/foo中数据存放于指定的my-volume中。

因此，VOLUME /foo在某些时候会产生歧义，如果不了解的话将导致问题。

