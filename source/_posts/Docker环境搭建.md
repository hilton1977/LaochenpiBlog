---
title: Docker环境搭建
date: 2018-08-20 14:11:30
tags:
    - 笔记
    - docker
---

>开发->部署测试->发布正式 在整体流程中每个人的开发环境可能各不相同、编译环境、运行环境。单机服务调整控制环境版本等可以保证发布一致性，但是如果当业务越来越庞大集群处理时需要部署多台机器时，可能每台机器的大大小小差异都会导致发布失败，处理起来非常麻烦。docker虚拟化来处理能保证发布环境一致性，可移植。通过docker 镜像你可以在任何版本linux服务器上进行发布。每个镜像就相当于个一个系统相互不影响独立环境。


![docker](http://www.ruanyifeng.com/blogimg/asset/2018/bg2018020901.png)

## 1.Docker 介绍
Docker 属于 Linux 容器的一种封装，提供简单易用的容器使用接口。它是目前最流行的 Linux 容器解决方案。

Docker 将应用程序与该程序的依赖，打包在一个文件里面。运行这个文件，就会生成一个虚拟容器。程序在这个虚拟容器里运行，就好像在真实的物理机上运行一样。有了 Docker，就不用担心环境问题。

总体来说，Docker 的接口相当简单，用户可以方便地创建和使用容器，把自己的应用放入容器。容器还可以进行版本管理、复制、分享、修改，就像管理普通的代码一样。

## 2.Docker安装
我的VPS用的Centos 7 那就用本版本记录搭建过程，docker的版本用CE社区版
``` bash
#下载yum-utils工具用于管理yum-config-manager可以配置源
yum install yum-utils
#添加docker-ce源
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
#查询docker-ce版本
yum list docker-ce --showduplicates | sort -r 
#指定安装18.06.0 版本
yum install dock-ce-18.06.0.ce
```
安装docker,默认是安装最高版本测试可以用，但是生产环境为了稳定尽量指定版本(stable稳定版)

## 3.Docker常用命令

``` bash
#启动docker服务
systemctl start docker
#自动启动docker服务
systemctl enable docker
#运行一个容器
docker run hello-world
#查看所有运行中容器
docker ps
#查看所有容器
docker ps -a
#停止容器
docker stop [name]
#查看镜像
docker images
#删除镜像
docker rmi [imageId]
#删除容器 
docker rm [name or id]
#进入某容器
docker attach [name or id]
#在运行中的容器中执行脚本 控制台交互  这里注意 有的镜像是 /bin/sh 
docker exec -itd [id or name] /bin/bash
```





