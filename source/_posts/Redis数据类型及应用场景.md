---
layout: post
title: Redis 数据类型及应用场景
date: 2019-04-10 16:19:30
categories: [Java]
tags:
   - redis
---

![](/images/redis.jpg)

> 记录学习 Redis 的数据类型以及实际项目中的运用场景

## Redis 存储类型
`Redis` 面试经常会闻到支持那些存储类型，常用的类型有`String`、`Hash`、`List`、`Set`、`Sorted Set`等。

### String
`   Strings` 数据类结构是最简单的 `Key-value`类型，`Value` 的值根据执行的命令可为数值或字符串。
> 常用命令: `set,get,decr,incr,mget` 等。

__应用场景__：例如计数器功能`incr`可以运用于接口调用次数每次调用增加+1，配合`decr`每次减少-1，执行完毕会返回操作之后的值。


## Hash
`Hash` 键值对格式，适合存放对象信息例如用户信息，用户名ID对应的用户信息`value`。有时候我们使用的是序列化对象取出和存在都需要序列化消耗性能。
> 常用命令： `hget,hset,hgetall` ，`Hash` 实现有2种在数据量较小时会采用类似一维数组紧凑存储,对应的`value`的`redisObject`的`encoding`为`zipmap`，当数据足够大时内部会自动转化成真正`HashMap`结构，`encoding`为`ht`。
  
__应用场景__：例如用户信息、后台列表信息、用户的权限信息等。

## List
`List` 链表，`Redis` 实现为一个双向链表，即可以反向查询和遍历，更方便操作但也带来更多的性能消耗。
> 常用命令: `lpush,rpush,lpop,rpop,lrange`等

__应用场景__：例如使用`lrange`可以做分页操作，一些列表信息展示，使用`ltrim`限制长度可限制最新N条数据。`lpsuh`、`rpush`添加数据，`lpop`、`rpop`删除数据。

## Set
`Set` 类似`List`的功能，主要功能可以去重当你不希望有重复数据可以使用，并且可以判断是否某数据是否在集合内还可以处理2个`Set`交集、并集、差集。
> 常用命令： `sadd,spop,smembers,sunion`等，实现方式是`value`永远为`null`的`HashMap`，通过计算`Hash`的方式来快速排重。

__应用场景__：例如用户权限公共的权限交集，微博里的共同好友、共同关注等。
