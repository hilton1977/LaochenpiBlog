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
#启动docker服务
systemctl start docker
#自动启动docker服务
systemctl enable docker
```
安装docker,默认是安装最高版本测试可以用，但是生产环境为了稳定尽量指定版本(stable稳定版)

## 3.Docker 镜像 容器
#### 镜像查询拉取
安装 docker 完毕，可以尝试安装一个镜像并运行，搜索镜像使用 `docker search [镜像名称]`,搜索的镜像 `OFFICAL` 标识的为官方镜像，其余的都是非官方人员自行构建的镜像并上传库共享。
![docker-search-alpine](/images/docker-search.png)
使用 `docker pull alpine` 下载拉取alpine镜像,然后使用`docker images` 查看镜像已有镜像，这里以`alpine`为模板
![docker-images](/images/docker-images.png)

#### 运行容器
基于alpine镜像启动一个容器 
``` bash
docker run -itd -p  8081:8081 --name myTest  alpine
```
- -i：以交互模式运行容器，通常与 -t 同时使用
- -d: 后台运行容器，并返回容器ID
- -t : 为容器重新分配一个伪输入终端，通常与 -i 同时使用
- -p: 端口映射，格式为：主机(宿主)端口:容器端口 8080端口的访问转发到容器的8080端口上
- --name: 为容器指定一个容器名
- alpine：这是指用 alpine 镜像为基础来启动容器。

启动完毕后 `docker ps` 查看正在运行的容器,  `docker ps -a` 查看容器。

#### 容器操作

``` bash
##### myTest 为容器名称 ##### 
#进行容器
docker attach myTest
#容器中执行脚本返回结果 (由于是alpine所以执行的)
docker exec -it myTest /bin/sh
#删除容器
docker rm myTest
#启动已有容器
docker start myTest
#停止容器
docker stop myTest
```
在容器中退出容器时需要注意的是通过`exit`返回宿主主机会导致容器直接停止并不是我们想要的结果，官方给出的退出容器并使其在后台继续运行使用 `ctrl+p+q` 安全退出不影响容器运行。 

## 4.DockerFile编写




