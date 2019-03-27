---
title: RabbitMQ 消息队列中间件
date: 2019-03-20 14:35:19
categories: [Java]
tags:
	 - 中间件
---
![](/images/RabbitMQ.jpg)

> RabbitMq消息队列中间件记录一些基本的概念和实际项目运用，消息队列常常会作为解决项目之间解耦的方案之一，特点异步消息可持久化不丢失高可用。实际项目中有各类场景可使用消息队列，例如发送邮件模块、业务消息通知、异步回调结果、日志信息的收集聚合等。

## RabbitMQ
RabbitMq 是一款由`erlang`开发实现 AMQP（Advanced Message Queueing Protocal）的开源消息中间件，消息中间件主要运用于组件之间解耦，消息发送者不需关心消息消费者的存在，AMQP 的主要特征是面向消息、队列、路由（点对点和发布/订阅）、可靠性、安全。

##### 安装
[Rabbit官方网站](http://www.rabbitmq.com/)进行下载，由于 RabbitMQ 由[ERLANG](http://www.erlang.org/downloads) 开发需要安装相关环境,具体版本查看官方文档。安装完毕可以通过http://127.0.0.1:15672 查看 RabbitMQ 管理中心，包含了 RabbitMQ 配置主题、队列、运行情况、连接等。
初始登陆账号：admin 密码：admin
![RabbitMQ管理中心](/images/RabbitMQ-Admin.png)

## SpringBoot 集成
SpringBoot 微服务项目集成 RabbitMQ 特别方便，`Maven`项目依赖添加`spring-boot-starter-amqp`依赖然后进行基本配置。
##### Maven 	依赖
``` xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```
##### yml 配置
``` properties
spring:
  rabbitmq:
  host: ip地址
  port: 端口号默认5672
  username: 用户名
  password: 密码
  publisher-confirms: 是否启动推送自动确认 true or false
  listener:
    direct:
	  acknowledge-mode: ack消息确认方式：auto 自动 manual 手动 none 不确认
```

> ACK机制就是为了保证数据一定被消费确认，默认配置为`auto`自动,在实际项目中如果消费者出现程序异常或者意外服务宕机会导致消息未消费但是ACK自动确认后，提供者并不知道消费者消息失败导致业务数据不一致。ACK 可以设置为手动 `manual`只有当消费者告诉中间件已经消费中间件才会吧这条消息删除掉,否者这条消息会一直在队列中存在直到消费者消息掉。
