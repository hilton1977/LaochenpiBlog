---
title: SkyWalking 搭建记录
date: 2019-04-02 11:08:24
categories: [Java]
tags:
	 - 中间件 
---

![](/images/SkyWalking.jpg)


## SkyWalking 
SkyWalking 是观察性分析平台和应用性能管理系统，提供分布式追踪、服务网格遥测分析、度量聚合和可视化一体化解决方案。它是一款国产开源软件源码在github上，
官方网站为 http://skywalking.apache.org/zh/ Github https://github.com/apache/incubator-skywalking/

![项目结构](/images/skywalkingFile.png)
#### agent 
探针项目里面包含`skywalking-agent.jar`