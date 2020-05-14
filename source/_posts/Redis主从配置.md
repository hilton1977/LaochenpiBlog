---
title: Redis 主从配置
tags:
  - Redis
categories:
  - 数据库
toc: false
date: 2019-11-12 14:54:18
---

![](/images/redis.jpg)


### Redis 主从架构
Redis 当项目规模越来越大承载需求越来越大单机不足以支撑当前业务，则需要`master-slave`主从架构来分担其压力承载更多`QPS`(__我一人无力承受，我需要更多的人来帮我承担，扛不住就摇人__)， 主写从读把压力分散给下面的小弟，更容易水平扩容支持更高并发。
![master-slave 架构](/images/redis-master-slave.png)

####  Replication 核心机制
- redis 采用异步方式复制数据，当从库进行复制时不会堵塞查询也不会影响主库，会使用旧数据提供服务，当数据复制完毕需删除旧数据加载最新数时无法提供服务。
- slave node 可以有其他 slave node 形成级联模式
- master node 可以有个多个 slave node
- slave 有且仅能有一个 master 节点

#### 主从复制流程
![主从复制流程](/images/redis-master-slave-replication.png)

1. slave 向 master 发送`psync`请求，初次连接 master 会触发全量复制`full resynchronization`，并把 master 的 `runid` 和 `offset` 发送给 slave 以便下一次增量复制
2. master 会 fork 子进程生成 RDB 快照并将后续执行的写命令写入 buffer（复制缓冲区）
3. master 把 RDB 快照和复制缓冲区发送给 slave，slave 会将 RDB 快照先存入本地磁盘然后再加载到内存中
4. 当网络波动导致复制失败重新连接会根据 slave 的`offset`与 master 的 `offset`对比看是否在复制缓冲区范围内判断是全量或增量复制

### Redis 主从配置
``` conf
################################# REPLICATION #################################
# 复制选项，slave复制对应的master。
# slaveof <masterip> <masterport>

# 如 master 设置了requirepass 则需要密码才能进行复制保证数据安全性
# masterauth <master-password>

# 当从库同主机失去连接或者复制正在进行
# 1) 如果slave-serve-stale-data设置为yes(默认设置) ：从库会继续响应客户端的请求。
# 2) 如果slave-serve-stale-data设置为no ：除去INFO和SLAVOF命令之外的任何请求都会返回一个错误”SYNC with master in progress”。
slave-serve-stale-data yes

# 是否开启子节点只读，一般开启只读用于主从读写分离
slave-read-only yes

# 是否开启无盘复制，在主从复制生产的 RDB 文件不在落地到磁盘而是直接通过 socket 传输到子节点适用于磁盘 IO紧张网络宽带充裕情况下使用，默认不开启 no
repl-diskless-sync no

# diskless 复制的延迟时间，防止设置为0。一旦复制开始，节点不会再接收新slave的复制请求直到下一个rdb传输。所以最好等待一段时间，等更多的slave连上来。
repl-diskless-sync-delay 5

# slave 根据指定的时间间隔向服务器发送ping请求。时间间隔可以通过 repl_ping_slave_period 来设置，默认10秒。
# repl-ping-slave-period 10

# 复制连接超时时间。master和slave都有超时时间的设置。master检测到slave上次发送的时间超过repl-timeout，即认为slave离线，清除该slave信息。slave检测到上次和master交互的时间超过repl-timeout，则认为master离线。需要注意的是repl-timeout需要设置一个比repl-ping-slave-period更大的值，不然会经常检测到超时。
# repl-timeout 60

#是否禁止复制tcp链接的tcp nodelay参数，可传递yes或者no。默认是no，即使用tcp nodelay。如果master设置了yes来禁止tcp nodelay设置，在把数据复制给slave的时候，会减少包的数量和更小的网络带宽。但是这也可能带来数据的延迟。默认我们推荐更小的延迟，但是在数据量传输很大的场景下，建议选择yes。
repl-disable-tcp-nodelay no

# 复制缓冲区大小，这是一个环形复制缓冲区，用来保存最新复制的命令。这样在slave离线的时候，不需要完全复制master的数据，如果可以执行部分同步，只需要把缓冲区的部分数据复制给slave，就能恢复正常复制状态。缓冲区的大小越大，slave离线的时间可以更长，复制缓冲区只有在有slave连接的时候才分配内存。没有slave的一段时间，内存会被释放出来，默认1m。
# repl-backlog-size 5mb

# master没有slave一段时间会释放复制缓冲区的内存，repl-backlog-ttl用来设置该时间长度。单位为秒。
# repl-backlog-ttl 3600

# 节点优先级，在哨兵模式下当主节点下线状态下会从子节点群众根据优先级选出一个新的主节点，当优先级为0时直接忽略
slave-priority 100

# redis提供了可以让master停止写入的方式，如果配置了min-slaves-to-write，健康的slave的个数小于N，mater就禁止写入。master最少得有多少个健康的slave存活才能执行写命令。这个配置虽然不能保证N个slave都一定能接收到master的写操作，但是能避免没有足够健康的slave的时候，master不能写入来避免数据丢失。设置为0是关闭该功能。
# min-slaves-to-write 3

# 延迟小于min-slaves-max-lag秒的slave才认为是健康的slave。
# min-slaves-max-lag 10
```

通过命令`slaveof no one`可去除主从关系，通过命令行`info replication`可以查询当前节点的复制信息
 ![复制信息](/images/info_replication.png)
<escape>
<table>
     <tr>
            <th>角色</th>
            <th>属性名</th>
            <th>属性值</th>
            <th>描述</th>
     </tr>
     <tr>
            <td rowspan="5">通用配置</td>
            <td>role</td>
            <td>master|slave</td>
            <td>节点角色</td>
     </tr>
     <tr>
           <td>repl_backlog_active</td>
           <td>1|0</td>
           <td>复制缓冲区状态 1 启动 0 未启动</td>
     </tr>
     <tr>
           <td>repl_backlog_size</td>
           <td>10465325</td>
           <td>复制缓冲区大小</td>
     </tr>
     <tr>
           <td>repl_backlog_first_byte_offset</td>
           <td>10465200</td>
           <td>复制缓冲区起始偏移量，标识当前缓冲区可用范围</td>
     </tr>     
     <tr>
           <td>repl_backlog_histlen</td>
           <td>10465200</td>
           <td>复制缓冲区已有数据长度（字节）</td>
     </tr>
     <tr>
            <td rowspan="3">主节点</td>
            <td>slave0</td>
            <td>slave0:127.0.0.1,port=6380,state=online,offset=5572,lag=0</td>
            <td>子节点基本信息</td>
     </tr>
     <tr>
            <td>connected_slaves</td>
            <td>1</td>
            <td>子节点数量</td>
     </tr>
     <tr>
           <td>master_repl_offset</td>
           <td>330652</td>
           <td>主节点复制偏移量</td>
     </tr>     
     <tr>
           <td rowspan="10">子节点</td>
           <td>master_host</td>
           <td>127.0.0.1</td>
           <td>主节点地址</td>
     </tr>     
     <tr>
           <td>master_port</td>
           <td>6379</td>
           <td>主节点端口号</td>
     </tr>     
     <tr>
           <td>master_link_status</td>
           <td>up|down</td>
           <td>主节点状态 up 上线 down 下线</td>
     </tr>     
     <tr>
           <td>master_last_io_seconds_ago</td>
           <td>-1|其他数字</td>
           <td>与主节点最后一次 io 通信时间间隔（秒）</td>
     </tr>     
     <tr>
           <td>master_link_down_since_seconds</td>
           <td>20</td>
           <td>主节点下线状态已过时间（秒）</td>
     </tr>     
     <tr>
           <td>master_sync_in_progress</td>
           <td>1|0</td>
           <td>子节点是否从主节点全量复制 1 是 0 否</td>
     </tr>     
     <tr>
           <td>slave_repl_offset</td>
           <td>330652</td>
           <td>子节点复制偏移量</td>
     </tr>
     <tr>
           <td>slave_priority</td>
           <td>100</td>
           <td>子节点优先级，在哨兵模式下会根据优先级重新选举主节点</td>
     </tr>
     <tr>
           <td>slave_read_only</td>
           <td>1|0</td>
           <td>子节点是否开启只读 1 是 0 否</td>
     </tr>
     <tr>
           <td>master_repl_offset</td>
           <td>330652</td>
           <td>主节点复制偏移量</td>
     </tr>
 </table>
 </escape>
 
##### 注意事项
- 合理设置 `repl-backlog-size` 复制缓冲区大小避免触发循环全量复制，当 master 接受新写入新请求速度大于 slave 同步的速度时，buffer 复制缓冲区内容还未被 slave 同步就被新写入数据所覆盖导致丢失部分指令数据，每一次增量复制 offset 偏移量不在缓冲区范围而触发循环全量复制
- 设置缓冲区超时 `client-output-buffer-limit slave 256mb 64mb 60` 大于256mb或者超过64mb 的状态且超过60s，根据实际情况配合 repl-timeout 同步超时进行调整使用
- slave 不会处理过期 key，通过 master 同步删除
- 从节点 `slave-read-only` 设置为 yes 只读状态
- 启无磁盘复制：`repl-diskless-sync` yes  快照将不落地磁盘直接通过网络复制避免IO性能差


### Sentinel 哨兵模式 
![哨兵模式架构](/images/redis-sentinel.png)
对于主从架构如果主节点宕机挂掉子节点将无法获取最新数据，对于这样的情况推出了哨兵模式当`master`挂掉后会从子节点重新选举一个新的`master`节点来保证高可用，旧`master`则会转化成新的`slave`节点
- **监控** ：sentinel 会不断检查`master`、`slave`服务器是否正常工作 
- **通知** ：当被监控服务出现问题会通过 Api 向管理人员发送通知
- **自动故障转移** ：当`master`失效无法提供服务时，将从`slaves`子节点群中选举一个新的`master`节点并将其他`slave`指向它

#### 哨兵配置
``` conf 
# 端口
port 26379

# 是否后台启动
daemonize yes

# pid文件路径
pidfile /var/run/redis-sentinel.pid

# 日志文件路径
logfile "/var/log/sentinel.log"

# 定义工作目录
dir /tmp

# 定义Redis主的别名, IP, 端口，2指的是需要至少2个Sentinel 认为主节点主观下线（subjectivly）才标记为客观下线（objectivly）并准备自动故障转移
sentinel monitor mymaster 127.0.0.1 6379 2

# 判断主观下线（subjectivly）最大超时时间
sentinel down-after-milliseconds mymaster 30000

# 并发同步新 master 节点数据的 slave 数量，数字越小当节点过多时需要完成 failover 的事件就越长，数字越大由于 replication 机制会有更多的 slave 无法提供服务
sentinel parallel-syncs mymaster 1

# 故障转移超时时间
sentinel failover-timeout mymaster 180000

# 不允许使用SENTINEL SET设置notification-script和client-reconfig-script。
sentinel deny-scripts-reconfig yes

# 故障通知执行相应的脚本，允许执行的最大事件为60秒超时kill，失败重试次数为10
sentinel notification-script <master-name> <script-path> 
```

启动哨兵后会__自动感知__到主节点下的所有从节点，通过`info`指令可知监控主节点的状况、地址、子节点数、监控哨兵数量
![哨兵情况](/images/sentinel-info.png)

> 每次故障转移会__重写__节点的配置文件，当之前__客观下线__的节点重新上线后会自动同步新主节点数据