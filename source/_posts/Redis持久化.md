---
title: Redis 持久化
tags:
  - Redis
categories:
  - 数据库
toc: false
date: 2019-02-22 10:27:08
---

![](/images/redis.jpg)

>记录学习Redis持久化，Redis为内存数据库当服务器异常关闭或重启会导致内存里的Redis数据丢失，Redis提供持久化方案来保证数据不丢失.

## Redis 持久化
Redis持久化有多种不同级别的方式
- `RDB` 持久化可以在指定时间范围内服务器生成数据集的`Snapshot` 时间点快照`point-in-time `(数据库中所有键值对数据)
- `AOF` 持久化记录服务器执行过的写操作命令，在服务启动通过执行命令来恢复数据集。`AOF`文件中的命令以Redis协议的格式保存，新命令会追加到文件末尾。
- `RDB` `AOF`同时使用，在Redis重启时优先使用`AOF`进行数据恢复，因为`AOF`的保存的数据通常比`RDB`文件所保存的数据更完整。
- 关闭持久化，数据仅在服务器运行时存在

## RDB 持久化-配置
- __save__ `save m n` (m 代表时间范围内 n 修改次数) 例如默认配置文件中的`save 900 1` 900秒内至少有一个Key发生变化则保存。
- __stop-writes-on-bgsave-error__ 默认值yes，当Redis后台保存失败时是否停止接受写操作。如果已经设置一些监控可选择关闭。
- __rdbcompression__ 默认值yes，对存储到磁盘的快照文件是否进行压缩(`LZF`算法压缩)，压缩会消耗一定CPU性能，具体根据实际情况设置是否压缩。
- __rdbchecksum__ 默认值yes，对于存储的快照文件使用`CRC64`算法进行数据校验，校验大概消耗10%的性能，如需大量性能可关闭跳过校验过程。
- __dbfilename__ 默认值 dump.rbd， 快照文件的名称。
- __dir__ 默认当前目录，快照文件的存放文件路径

#### RDB 优点
- Redis在保存RDB会fork出子进程进行，几乎不影响Redis处理效率。
- RDB非常适合灾难恢复（`disaster reconvery`），每次快照会生成完整的快照文件，可根据业务需求进行多备云备份。
- RDB在恢复大数据集时比AOF速度更快。 

#### RDB 缺点
- RDB快照是定期生成，在时间范围内服务发生宕机可能导致会丢失部分数据
- RDB在大数据快照生成上会消耗大量CPU性能，如CPU性能不足或紧张时会影响Redis对外服务。

## AOF 持久化
__AOF__（`append-only file`）持久化:将 Redis 执行的每一条写请求都记录在一个日志文件里，在 Redis 启动后会执行所有的写操作达来恢复数据。AOF 默认是关闭状态，AOF 提供3种fsync配置
- __appendfsync no__ 不进行fsync，由OS来决定什么时候进行同步，速度最快
- __appendfsync always__ 每一次操作都进行fsync，速度较慢
- __appendfsync everysec__ 折中的做法，交由后台线程每秒fsync一次

#### AOF 优点
数据更安全，在配置 appendfsync always或 appendfsync everysec会及时把每条执行的写操作都记录都追加到AOF文件末尾即使是服务出现故障至多损失1s之内的数据。

#### AOF 缺点

##### 参考地址
http://doc.redisfans.com/topic/persistence.html
https://baijiahao.baidu.com/s?id=1611955931705092609&wfr=spider&for=pc