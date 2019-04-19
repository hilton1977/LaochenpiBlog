---
title: RabbitMQ 消息队列中间件
date: 2019-03-20 14:35:19
categories: [Java]
tags:
	 - 中间件
---
![](/images/RabbitMQ.jpg)

> RabbitMq消息队列中间件记录一些基本的概念和实际项目运用，消息队列常常会作为解决项目之间解耦的方案之一，特点异步消息可持久化不丢失高可用。实际项目中有各类场景可使用消息队列，例如发送邮件模块、业务消息通知、异步回调结果、日志信息的收集聚合等。

### RabbitMQ
RabbitMq 是一款由`erlang`开发实现 AMQP（Advanced Message Queueing Protocal）的开源消息中间件，消息中间件主要运用于组件之间解耦，消息发送者不需关心消息消费者的存在，AMQP 的主要特征是面向消息、队列、路由（点对点和发布/订阅）、可靠性、安全。

#### Rabbit 基本概念
- __Message__ 
由消息头和消息体组成，消息头里包含了`routeKey`用于标记消息需要发送到那个队列，`priority`消息优先权，`delivery-mode`消息可能需要持久性存储。
- __Channel__
信道，多路复用连接中的一条独立的双向数据流通道，是建立在`TCP`连接内的虚拟连接，发布消息、订阅队列、接受消息等都是通过信道完成，当你的项目连接上RabbitMq你可以在控制台`Channel`中发现一条新的记录包含了信道的各类信息，当项目停止后该条信道就会销毁。
![Channel信息](/images/RabbitMq-Channel.png)
- __Exchanges__
交换器，消息发送到交换器根据配置的规则再转发到相应队列，转发的规则有4中分别为`direct`、`topic`、`fonout`、`headers`。

#### Exchanges 路由规则
- __direct__
属于最常见的一种转发规则，单播模式根据`routeKey`匹配唯一的队列进行发送。
![Direct](/images/Exchanges-direct.jpg)
- __topic__
主题模式通过`#`（匹配0个或多个单词）和`*`（匹配一个单词）来绑定队列，如果不使用通配符可作为普通的`direct`队列，根据实际情况灵活配置。
![Topic](/images/Exchanges-topic.jpg)
- __fanout__ 
广播模式，每个发送到交换器上的信息都会被转发到绑定到该交换器上的所有队列，`fanout`类型的转发消息是最快的。
![fanout](/images/Exchanges-fanout.jpg)


#### 安装
[Rabbit官方网站](http://www.rabbitmq.com/)进行下载，由于 RabbitMQ 由 [ERLANG](http://www.erlang.org/downloads) 开发需要安装相关环境,具体版本查看官方文档。安装完毕可以通过http://127.0.0.1:15672 查看 RabbitMQ 管理中心，包含了 RabbitMQ 配置主题、队列、运行情况、连接等。

初始登陆账号：admin 密码：admin
![RabbitMQ管理中心](/images/RabbitMQ-Admin.png)

### SpringBoot 集成
SpringBoot 微服务项目集成 RabbitMQ 特别方便，`Maven`项目依赖添加`spring-boot-starter-amqp`依赖然后进行基本配置，添加`@EnableRabbit`开启 RabbitMQ 自动配置，自动装配的源码可以查看`RabbitAutoConfiguration`类。
#### Maven 依赖
``` xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```
#### yml 配置
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
     simple
       acknowledge-mode: ack消息确认方式：auto 自动 manual 手动 none 不确认

```

`ACK`机制就是为了保证数据一定被消费确认，默认配置为`auto`自动,在实际项目中如果消费者出现程序异常或者意外服务宕机会导致消息未消费但是`ACK`自动确认后，提供者并不知道消费者消费失败导致业务数据不一致。`ACK` 可以设置为手动 `manual`只有当消费者告诉中间件已经消费中间件才会把这条消息从队列中删除,否者这条消息会一直在队列中存在直到消费者重新消息掉。

#### 提供者
``` java
package com.example.rabbitmq.rabbitProvider;

import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.stereotype.Component;

import javax.annotation.Resource;

@Component
public class RabbitMqProvider {

    @Resource
    private RabbitTemplate rabbitTemplate;

    public void provider(){
        rabbitTemplate.convertAndSend("myTestQueue","测试测试123");
    }

}
```
这里使用`RabbitTemplete`进行发送消息，已封装了各类常用的推送消息方法，`myTestQueue`为队列名称。

#### 消费者 Ack 确认
``` java 
package com.example.rabbitmq.mytest;

import com.rabbitmq.client.Channel;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.annotation.RabbitHandler;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;
import java.io.IOException;

@Component
@RabbitListener(queues = "myTestQueue")
public class RabbitMqConsumer {

@RabbitHandler()
public void consumer(String msg, Channel channel, Message message) {
    try {
        System.out.println("接受消息" + msg);
        channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```
上述代码使用了`Channel`进行`Ack`确认，队列中有无数条信息为了确认唯一性，调用`basicAck`方法进行确认，`message.getMessageProperties().getDeliveryTag()`获取消息的唯一`tag`值。