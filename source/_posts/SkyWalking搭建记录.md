---
title: SkyWalking 搭建记录
date: 2019-04-02 11:08:24
categories: [Java]
tags:
	 - 中间件 
---

![](/images/SkyWalking.jpg)


## SkyWalking 
`SkyWalking` 是观察性分析平台和应用性能管理系统，提供分布式追踪、服务网格遥测分析、度量聚合和可视化一体化解决方案。它是一款国产开源软件源码在github上，
官方网站为 http://skywalking.apache.org/zh/ Github https://github.com/apache/incubator-skywalking/

![项目结构](/images/skywalkingFile.png)
### Agent 
`Agent`探针项目里面包含`skywalking-agent.jar`，使用`JavaAgent`做字节码侵入，对代码无侵入，收集格式化数据然后通过`Http`或`gRpc`方式发送到`SkyWalking Collector`。
通过`agent.config`配置探针
- agent.application_code 应用的唯一标识，用于UI展示App的名称
- collector.servers 用于配置收集的数据信息发送到`collector`地址
- logging.level 日志的收集级别

##### 集成方式
常用的`Java`项目运行方式`Jar`方式或者`War`通过容器启动，例如`Tomcat`
- __Jar__ 配置JVM运行添加探针 `-javaagent:[skywalking-agent.jar绝对路径]`
- __Tomcat__ `tomcat`目录`bin`下的`catalina`脚本
``` shell
Window setlocal添加 
set JAVA_OPTS=-javaagent:[skywalking-agent.jar绝对路径]

Linux
CATALINA_OPTS="$CATALINA_OPTS -javaagent:[skywalking-agent.jar绝对路径]"; export CATALINA_OPTS
```

### Collector
`Collector`