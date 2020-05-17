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

> 集群搭建完毕后可以通过`redis-cli -c`集群模式进入客户端，配置文件中`cluster-config-file nodes.conf`就是用来保存节点交换的各种信息，通过`cluster nodes`查看整个集群节点数量、状态、负责的哈希槽范围，`cluster info`集群信息

![集群](/images/2020/05/17/97ae0d90-980b-11ea-8fab-19b1e8901849.png)

- **cluster_state** 集群状态（**当集群某个节点失效时根据 cluster-require-full-coverage 设置看到的状态可能不一样**）
- **cluster_slots_assinged** 集群哈希槽数量
- **cluster_slots_ok** 正常的哈希槽数
- **cluster_slots_pfail** 主观失败哈希槽数量
- **cluster_slots_fail** 客观失败哈希槽数量
- **cluster_knonw_nodes** 集群中节点数
- **cluster_size** 集群分片数

#### 集群测试
![集群存入](/images/2020/05/17/50370000-980d-11ea-879e-43df0e436c7d.png)
当我们存入或获取`key`时客户端会通过算法得到对应的哈希槽，如果当前节点不负责该哈希槽则会查询哈希槽路由表查找对应负责的节点并重定向保存

### 集群脚本

#### 新增子节点
 为保证整个集群高可用，需要给各个哈希槽负责节点分配至少一个从节点用于故障转移，使用脚本给节点添加从节点，注意该节点必须启动状态且无数据和集群信息否则无法添加
``` bash
# 添加到主节点为master-id作为子节点
redis-trib.rb add-node --slave --master-id [主节点 node-id] [新增节点 host:port] [集群中任意节点 host:port]  
```
#### 新增主节点
扩容 当数据量越来越大时可以随时进行动态扩容，加入新的节点并分配部分哈希槽会让你选择哈希槽的数量和接受节点
``` bash
# 添加主节点
redis-trib.rb add-node [新增节点 host:port] [集群中任意节点 host:port]  
# 迁移哈希槽到新节点 
redis-trib.rb reshard [集群中任意节点 host:port]
```
> 如果发现有某个哈希槽有问题，使用`redis-trib.rb fix [集群节点 host:port]`进行修复，如果失败则需要登录有问题的节点
使用`cluter setslot [哈希槽号] stable`取消迁移
