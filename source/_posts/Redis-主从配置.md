---
title: Redis 主从配置
tags: []
categories: []
toc: false
date: 2019-11-12 14:54:18
---

![](/images/redis.jpg)


## Redis 主从架构
Redis 当项目规模越来越大承载需求越来越大单机不足以支撑当前业务，则需要`master-slave`主从架构来分担其压力承载更多`QPS`(__我一人无力承受，我需要更多的人来帮我承担，扛不住就摇人__)， 主写从读把压力分散给下面的小弟，更容易水平扩容支持更高并发。
![master-slave 架构](/images/redis-master-slave.png)

### redis replication 核心机制
- redis 采用异步方式复制数据，当从库进行复制时不会堵塞查询也不会影响主库，会使用旧数据提供服务，当数据复制完毕需删除旧数据加载最新数时无法提供服务。
- slave node 可以有其他 slave node 形成级联模式
- master node 可以有个多个 slave node

#### 主从复制流程
![主从复制流程](/images/redis-master-slave-replication.png)

1. slave 向 master 发送 psync 请求，初次连接 master 会触发全量复制 full resynchronization，并把 master 的 runid 和 offset 发送给 slave以便下一次增量复制
2. master 会 fork 子进程生成 RDB 快照并将后续执行的写命令写入 buffer（复制缓冲区）
3. master 把 RDB 快照和复制缓冲区发送给 slave，slave 会将 RDB 快照先存入本地磁盘然后再加载到内存中
4. 当网络波动导致复制失败重新连接会根据 slave 的 offset 是否在复制缓冲区范围内来进行全量复制或增量复制

#### 注意事项
- 合理设置 repl-backlog-size 复制缓冲区大小避免触发循环全量复制，当 master 接受新写入新请求速度大于 slave 同步的速度时，buffer 复制缓冲区内容还未被 slave 同步就被新写入数据所覆盖导致丢失部分指令数据，每一次增量复制 offset 偏移量不在缓冲区范围而触发循环全量复制
- 设置缓冲区超时 client-output-buffer-limit slave 256mb 64mb 60 大于256mb或者超过64mb 的状态且超过60s，根据实际情况配合 repl-timeout 同步超时进行调整使用
- slave 不会处理过期 key，通过 master 同步删除
- 从节点 slave-read-only 设置为 yes 只读状态
- 重启 master node 会 导致 runid 重新生成，当 slave node 请求同步会发现 runid 发生改变会进行全量复制 

#### Redis 配置
``` conf
# 配置 主节点 masterip masterport
slaveof <masterip> <masterport>
```

