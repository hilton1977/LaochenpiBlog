---
title: Docker 环境搭建
date: 2018-08-20 14:11:30
categories: [技术]
tags:
    - Java
---

>开发->部署测试->发布正式 在整体流程中每个人的开发环境可能各不相同、编译环境、运行环境。单机服务调整控制环境版本等可以保证发布一致性，但是如果当业务越来越庞大集群处理时需要部署多台机器时，可能每台机器的大大小小差异都会导致发布失败，处理起来非常麻烦。docker虚拟化来处理能保证发布环境一致性，可移植。通过docker 镜像你可以在任何版本linux服务器上进行发布。每个镜像就相当于个一个系统相互不影响独立环境。


![docker](http://www.ruanyifeng.com/blogimg/asset/2018/bg2018020901.png)

## 1.Docker 介绍
Docker 属于 Linux 容器的一种封装，提供简单易用的容器使用接口。它是目前最流行的 Linux 容器解决方案。

Docker 将应用程序与该程序的依赖，打包在一个文件里面。运行这个文件，就会生成一个虚拟容器。程序在这个虚拟容器里运行，就好像在真实的物理机上运行一样。有了 Docker，就不用担心环境问题。

总体来说，Docker 的接口相当简单，用户可以方便地创建和使用容器，把自己的应用放入容器。容器还可以进行版本管理、复制、分享、修改，就像管理普通的代码一样。

## 2.Docker 安装
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
<<<<<<< HEAD
=======
```
安装docker，默认是安装最高版本测试可以用，但是生产环境为了稳定尽量指定版本(stable稳定版)

## 3.Docker 常用命令

``` bash
>>>>>>> 89e26ef27826ebc8a8b06014969c8ead3bb74a3e
#启动docker服务
systemctl start docker
#自动启动docker服务
systemctl enable docker
```
安装docker,默认是安装最高版本测试可以用，但是生产环境为了稳定尽量指定版本(`stable稳定版`)

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
- alpine：这是指用 `alpine` 镜像为基础来启动容器。

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

## 4.DockerFile
Dockerfile 是一个文本文件，其内包含了一条条的指令`(Instruction)`，每一条指令构建一层，因此每一条指令的内容，就是描述该层应当如何构建。我们可以根据实际的开发需求通过`dockerfile`来自定义镜像，JUST DO IT！

##### FROM 指令
FROM <image\>:<tag\> 相当于建造一个大楼地基的选择，选择不同的地基来搭建不一样的大楼。
- 操作系统类基础搭建例如 `ubuntu`、`dabin`、`centos`
- 开发语言作为基础搭建例如`java`、`nodejs`、`python` 
- 服务类镜像作为基础 `oralce`、`mysql`、`nginx`、`tomcat`
- 自定义混合类作为基础在其他自定义环境镜像基础上搭建

所有的镜像地基都可以从docker库中拉取，选择合理的基础镜像可以让你更快的去构建你的镜像，省心省力。

#### RUN 指令
RUN 就像是执行shell指令，常常用于更新安装需要的生产软件服务等。RUN有2种执行方式
- shell 格式： RUN <命令> ，就像直接在命令行中输入的命令一样：`RUN apt-get --update`
- exec 格式： RUN ["可执行文件", "参数1", "参数2"]：`RUN ["apt-get","--update"]`


__注意__：多行命令不要写多个`RUN`，原因是`Dockerfile`中每一个指令都会建立一层.多少个`RUN`就构建了多少层镜像，会造成镜像的臃肿多层，不仅仅增加了构件部署的时间，还容易出错。`RUN`书写时的换行符是`\`，记得下载压缩软件操作完毕后`rm`不必要的软件压缩包和缓存让镜像更精简。

#### CMD 指令
`CMD` 指令的格式和 `RUN` 相似也是两种格式，`CMD` 执行脚本在`dockerfile`只能存在一条，多条只执行最后一条，当有多个时只会执行最后一个，一般用于执行开启某些服务 `tomcat`、`oracle`、`nginx`等。

#### ENTRYPOINT 指令
`ENTRYPOINT` 执行脚本在`dockerfile`只能存在一条，多条只执行最后一条，容器启动后执行且不会被`docker run`提供的参数覆盖。

### RUN  ENTRYPOINT  CMD 小结
- `CMD` 和 `ENTRYPOINT` 推荐使用`Exec`格式，因为指令可读性更强，更容易理解。`RUN` 则两种格式都可以。
- `RUN`用来执行脚本构建基础镜像，`CMD` `ENTRYPOINT` 用来构建完镜像容器启动后执行一些操作。
- `CMD` 会被`docker run` 后的执行脚本覆盖不执行，`ENTRYPOINT` 则不会被覆盖始终会被执行，如果需要覆盖运行需要`–entrypoint`参数。
- `ENTRYPOINT` 和 `CMD` 同时存在时谁在最后谁能执行，`CMD` 可作为 `ENTRYPOINT` 的执行参数灵活配合使用。

#### COPY 指令
用于从上下文路径复制文件到容器目标路径中，`copy package.json /usr/src/app/` 把`package.json`复制到容器 `/usr/src/app`路径下
- COPY <源路径>... <目标路径>
- COPY ["<源路径>"，......，"<目标路径>"]  `......`代表若干源路径

#### ADD 指令
`ADD` 指令和 `COPY` 的格式和性质基本一致，是在 `COPY` 基础上增加了一些功能。比如`<源路径>`可以是一个`URL`，这种情况下 Docker 引擎会试图去下载这个链接的文件放到`<目标路径>`去。如果`<源路径>`为一个` tar` 压缩文件的话，压缩格式为`gzip` , `bzip2` 以及 `xz` 的情况下，`ADD`指令将会自动解压缩这个压缩文件到`<目标路径>`去。`ADD` 指令会令镜像构建缓存失效，从而可能会令镜像构建变得比较缓慢，`ADD` 还包含了一些复杂的的功能其行为也不一定清晰，所以官方推荐使用`COPY`来进行文件的复制。

#### ENV 指令
`ENV` 用于设置环境变量在后续的指令可以直接引用
- ENV <key\> <value\>
- ENV <key1\>=<value1\> <key2\>=<value2\>...

##### Docker build构建
所有的脚本编写完毕使用`docker bulid` 对 Dockerfile 进行构建，详细的命令如下
``` bash

```

``` bash
#基于镜像 这里使用alpine 主要是体积小构建速度更快
FROM alpine
#构建维修者 
MAINTAINER 285635652@qq.com
RUN apt-get update /
  && apt-get java
```



