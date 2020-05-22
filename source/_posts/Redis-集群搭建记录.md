---
title: Redis 集群搭建记录
tags:
  - Redis
categories:
  - 数据库
toc: false
date: 2020-05-17 14:06:04
---

![](/images/redis.jpg)
> 记录下本地 windows 搭建集群全过程

### Redis 环境准备
- 安装 [**Redis for window**](https://github.com/MicrosoftArchive/redis/releases) 微软开源，不过比官方的版本落后很多，生产肯定是用`linux`版本
- Redis 官方提供`Ruby`脚本自动化建立集群，根据自身系统下载 64 或 32 位 [**RubyInstaller**](https://rubyinstaller.org/)
- 安装好`Ruby`环境后通过命令行`gem install redis` 安装相关脚本依赖
	``` bash
	C:\Users\28563>gem install redis
	Successfully installed redis-4.1.4
	Parsing documentation for redis-4.1.4
	Done installing documentation for redis after 1 seconds
	1 gem installed
	```

- 由于`window`版本`reids`无脚本，需要下载相应版本的`redis`提取出 [**redis-trib.rb**](https://github.com/antirez/redis/releases)

### 节点创建
- 复制默认的配置文件并修改以下参数
    ``` conf
    # 端口号
    port 6380
    # rdb 存放地址
    dbfilename dump-6380.rdb
    # 日志文件
    logfile "server_log_6380.txt"
    # 开启集群
    cluster-enabled yes
    # 集群节点信息保存地址
    cluster-config-file nodes-6380.conf
    # 集群节点超时时间
    cluster-node-timeout 15000
    # 集群某哈希槽节点失效继续提供服务
    cluster-require-full-coverage no
    ```
- 启动相关节点`redis-server [配置文件.conf]`，由于是集群模式启动后还未分配`hash slot`所以无法使用
- 启动脚本`ruby redis-trib.rb create [host:port]`**至少3个节点且不能已有数据**才可以创建成功，执行完毕会根据节点数量自动分配`hash slot`

#### 集群信息
集群搭建完毕后可以通过`redis-cli -c`集群模式进入客户端，配置文件中`cluster-config-file nodes.conf`就是用来保存节点交换的各种信息，通过`cluster nodes`查看整个集群节点数量、状态、负责的哈希槽范围，`cluster info`集群信息

``` bash
# 集群基本信息
127.0.0.1:6379> cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:3
cluster_size:3
cluster_current_epoch:7
cluster_my_epoch:5
cluster_stats_messages_sent:41100
cluster_stats_messages_received:41061

# 集群节点信息
127.0.0.1:6379> cluster nodes
da808f4f9b33566baffd36b78b75c766a61c53c9 127.0.0.1:6380 master - 0 1589784070344
 6 connected 0-499 5961-10922
7ba092b079f54886a4bdbfbaae4bbf161f131102 127.0.0.1:6381 master - 0 1589784069344
 7 connected 500-998 11422-16383
68448d769184298cb3678415ecec156e0f88c1cc 127.0.0.1:6379 myself,master - 0 0 5 co
nnected 999-5960 10923-11421
```

- **cluster_state** 集群状态（**当集群某个节点失效时根据 cluster-require-full-coverage 设置看到的状态可能不一样**）
- **cluster_slots_assinged** 集群哈希槽数量
- **cluster_slots_ok** 正常的哈希槽数
- **cluster_slots_pfail** 主观失败哈希槽数量
- **cluster_slots_fail** 客观失败哈希槽数量
- **cluster_knonw_nodes** 集群中节点数
- **cluster_size** 集群分片数

#### 集群测试
``` bash
127.0.0.1:6379> set k1 1
-> Redirected to slot [12706] located at 127.0.0.1:6381
OK
127.0.0.1:6381> set k2 2
-> Redirected to slot [449] located at 127.0.0.1:6380
OK
127.0.0.1:6380> set k3 3
-> Redirected to slot [4576] located at 127.0.0.1:6379
OK 
```
当我们存入或获取`key`时客户端会通过算法得到对应的哈希槽，如果当前节点不负责该哈希槽则会查询哈希槽路由表查找对应负责的节点并重定向保存

### 集群脚本
#### 集群信息
- 通过`info`可知每个节点已有`key`数量、哈希槽数、从节点数量
    ``` bash
    # 查看集群情况
    ruby redis-trib.rb info [集群节点 host:port]
    127.0.0.1:6379 (68448d76...) -> 0 keys | 6460 slots | 0 slaves.
    127.0.0.1:6380 (da808f4f...) -> 0 keys | 4962 slots | 0 slaves.
    127.0.0.1:6381 (7ba092b0...) -> 0 keys | 4962 slots | 0 slaves.
    127.0.0.1:6382 (cd8124ba...) -> 0 keys | 0 slots | 0 slaves.
    [OK] 0 keys in 4 masters.
    0.00 keys per slot on average.
    ```
- 通过`check`可知每个节点`node-id`、地址、哈希槽分布和数量、额外副本数量，所有哈希槽是否在
    ``` bash
    # 检查集群情况
    ruby redis-trib.rb check [集群节点 host:port]
    M: 68448d769184298cb3678415ecec156e0f88c1cc 127.0.0.1:6379
       slots:0-5960,10923-11421 (6460 slots) master
       0 additional replica(s)
    M: da808f4f9b33566baffd36b78b75c766a61c53c9 127.0.0.1:6380
       slots:5961-10922 (4962 slots) master
       0 additional replica(s)
    M: 7ba092b079f54886a4bdbfbaae4bbf161f131102 127.0.0.1:6381
       slots:11422-16383 (4962 slots) master
       0 additional replica(s)
    M: cd8124bab703fba65d7ba91330b72c0e89072269 127.0.0.1:6382
       slots: (0 slots) master
       0 additional replica(s)
    [OK] All nodes agree about slots configuration.
    >>> Check for open slots...
    >>> Check slots coverage...
    [OK] All 16384 slots covered.
    ```


#### 新增子节点
 为保证整个集群高可用，需要给各个哈希槽负责节点分配至少一个从节点用于故障转移，使用脚本给节点添加从节点，注意该节点必须启动状态且无数据和集群信息否则无法添加，新增的主节点默认是没有哈希槽，需要通过迁移命令转移部分哈希槽
``` bash
# 添加到主节点为master-id作为子节点
ruby redis-trib.rb add-node --slave --master-id [主节点 node-id] [新增节点 host:port] [集群中任意节点 host:port]  
```

#### 删除节点
从节点可直接删除，主节点如果有哈希槽则需要将拥有的哈希槽迁移到其他节点才能删除，删除节点会自动关闭
``` bash
# 删除节点
ruby redis-trib.rb del-node [删除节点 host:port] [删除节点 node-id]
>>> Removing node cd8124bab703fba65d7ba91330b72c0e89072269 from cluster 127.0.0.
1:6382
>>> Sending CLUSTER FORGET messages to the cluster...
>>> SHUTDOWN the node.
```

#### 迁移哈希槽
迁移可以对已有节点进行扩容或缩容，从一个或多个源节点指定哈希槽数量迁移到目标节点，
如果发现有某个哈希槽有问题，使用`redis-trib.rb fix [集群节点 host:port]`进行修复，如果失败则需要登录有问题的节点，使用`cluter setslot [哈希槽号] stable`取消迁移
``` bash 
# 迁移哈希槽到新节点
ruby redis-trib.rb reshard -- 额外参数 [集群节点 host:port]
```
##### 额外参数
- **from** [源节点 node-id 多个逗号隔开]
- **to** [目标节点 node-id]
- **slots** [哈希槽数量]
- **yes** [迁移无序手动确认]
- **pipeline** [批量迁移数量]
- **timeout** [控制每次migrate操作的超时时间，默认为60000毫秒]
#### 平衡哈希槽
当节点越来越多时有的哈希槽多有的哈希槽少可以通过命令平衡每个节点哈希槽数量
``` bash
# 平衡集群哈希槽
ruby redis-trib.rb rebalance -- 额外参数 [集群节点 host:port]
```
##### 额外参数
- **weight** [集群节点权重 node-id = 权重 多个空格隔开 默认权重1] 每个节点分配的哈希槽计算方法为：**slot数量 * （节点权重 / 总权重）**
- **threshold** [节点哈希槽阈值大于才参与平衡]
- **pipeline** [平衡迁移批量数量]
- **use-empty-masters** [使用空哈希槽的主节点参与平衡]
- **simulate** [模拟平衡并不会真正执行迁移操作]
- **timeout** [设置migrate命令的超时时间]

#### 集群执行命令
当集群节点越来越多时可通过`call`指令在每个集群节点都执行相应在每个节点都执行某命令
``` bash
# 集群执行指令
ruby redis-trib.rb call [集群节点 host:port] [redis 指令]
# 执行 cluster info 得到结果
>>> Calling CLUSTER info
127.0.0.1:6379: cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:3
cluster_size:3
cluster_current_epoch:7
cluster_my_epoch:5
cluster_stats_messages_sent:40322
cluster_stats_messages_received:4028
```