---
layout: post
title: nginx对spring boot项目进行代理
category: 默认分类
tags: []
keywords: 
---

#nginx对spring boot项目进行代理/n

使用nginx对spring boot项目进行代理
==========================================================================================

> 摘要：使用nginx对spring boot项目进行反向代理，并且使用轮询均衡负载策略

均衡负载与集群
-------

集群和均衡都涉及到多个机器提供的服务的问题

不同点是，集群是互相通信、协同的的多个服务，服务之前能够状态共享。而均衡负载一般说的是，服务之间相互独立，不知道彼此。因此，使用均衡负载最好是提供的无状态的服务，如果服务有状态，那么就需要一个统一管理状态的服务单独部署

搭建过程
----

### 相关工具

*   使用spring boot快速搭建一个web项目
*   virtual box作为虚拟机，并安装docker
*   独立安装nginx，对docker容器进行反向代理

### spring boot项目

只需要添加`spring-boot-starter-web`依赖即可，并且添加如下代码

    package com.luzj.mychdocker10;
    
    import org.springframework.web.bind.annotation.RequestMapping;
    import org.springframework.web.bind.annotation.RestController;
    
    /\*\*
    
    - @author luzj
    - @description:
    - @date 2018/7/27
      \*/
      @RestController
      public class IndexController {
      @RequestMapping("/index")
      public String index(){
      return "hello docker 1";
      }
    
          @RequestMapping("/tips")
          public String tips(){
              return "是莉娜呀!!!";
          }
    
      }

然后将项目打包，之后修改上面`index()`的代码，使返回不同的字符串，这样便于观察nginx是否将请求分发到每一个服务上面去

**服务2代码**

      @RequestMapping("/index")
          public String index(){
              return "hello docker 2";
          }

**服务3代码**

      @RequestMapping("/index")
          public String index(){
              return "hello docker 3";
          }

分别将3个服务打成`jar`，待之后部署到docker上，分别是`mychdocker10-0.0.1-SNAPSHOT.jar`,`mychdocker10-0.0.2-SNAPSHOT.jar`,`mychdocker10-0.0.3-SNAPSHOT.jar`

### docker+nginx部署项目

docker和nginx全部部署在virtual box上。其中docker部署用来部署前面的`jar`包，而nginx单独部署，对三个docker容器代理。

之所以nginx单独部署，是因为单独拉取的nginx镜像无法使用vim编写配置文件。

**使用docker部署jar包**

    # 在 /home/<username>/目录下新建 dockerdir目录
      mkdir -p /home/<username>/dockerdir
    
    # 将 jar 包拷入 dockerdir 中
    
    # 编写 Dockerfile
    
    vim Dockerfile
    
    # Dockerfile 内容
    
    FROM java:8
    add mychdocker10-0.0.3-SNAPSHOT.jar app.jar
    expose 8080
    entrypoint ["java","-jar","/app.jar"]
    
    # 编译 docker 镜像,balancejar 为镜像名称, . 表示当前目录
    
    docker build -t balancejar .
    
    # 运行容器，8081:8080 表示将 docker 内部 8080 端口映射到宿主机的 8081 端口，docker 的宿主机就是 virtual box，之后还需要对 virtual box 再进行一次映射，映射到本机
    
    docker run -d -p 8081:8080 --name balanceContainer balancejar

另外两个jar的的部署方式和上面的一样，分别映射到8082和8083端口

docker镜像列表：  
![image](https://raw.githubusercontent.com/dangshanli/gallery/master/docker/docker%E9%95%9C%E5%83%8F.png)

docker容器列表：  
![imags](https://raw.githubusercontent.com/dangshanli/gallery/master/docker/docker%E5%AE%B9%E5%99%A8.png)

### virtual box端口映射

我们的docker是安装在virtual box上面的，如果想让本机可以访问，还需要将虚拟机的端口映射到宿主机的端口

选择`[控制]->[设置]->[网络]->[端口转发]`，即可见端口映射配置面板，如下图

![image](https://raw.githubusercontent.com/dangshanli/gallery/master/docker/%E7%AB%AF%E5%8F%A3%E8%BD%AC%E5%8F%91.png)

### 查看docker部署结果

![image](https://raw.githubusercontent.com/dangshanli/gallery/master/docker/8081%E7%AB%AF%E5%8F%A3%E5%B1%95%E7%A4%BA.png)  
可以看到直接在本机上可以访问了

### 配置nginx

**nginx打开配置文件**

    vim /usr/local/nginx/conf/nginx.conf

**添加如下配置信息到http模块里面**

    # 配置代理server组，标识为Tomcat
    upstream tomcat {
          server  127.0.0.1:8081 weight=10;
          server  127.0.0.1:8082 weight=10;
          server  127.0.0.1:8083 weight=10;
    }
    
    server {
    listen 80; # 默认监听端口 80
    server_name cc520.me;# 对外服务名
    
    # ...其他配置信息
    
    # 配置代理路径
    
    location /docker{
    proxy_pass http://tomcat/;
    }
    
            # 对“/”路径转发 /docker
            location = / {
            return 302 /docker;
            }
        # ...其他配置信息
    
    }
    
    

**映射nginx监听端口**

将virtual box 80端口映射到本机

### 访问nginx进行验证

依次访问`http://cc520.me/docker/index`3次，可以看到如下结果

**第一次**  
![image](https://raw.githubusercontent.com/dangshanli/gallery/master/docker/%E4%BB%A3%E7%90%861.png)

**第二次**  
![image](https://raw.githubusercontent.com/dangshanli/gallery/master/docker/%E4%BB%A3%E7%90%862.png)

**第三次**  
![image](https://raw.githubusercontent.com/dangshanli/gallery/master/docker/%E4%BB%A3%E7%90%863.png)

可以看到nginx使用默认的**轮询策略**进行服务分发，访问3次，依次访问了代理的3个服务

小结
--

我们使用docker部署springboot项目，之后使用nginx对3个docker容器进行代理，最后在浏览器访问查看nginx是否对springboot项目进行均衡负载

