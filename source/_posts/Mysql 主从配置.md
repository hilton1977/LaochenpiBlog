---
layout: post
title: MySql 主从集群配置
date: 2019-03-25 09:07:05
categories: [数据库]
tags:
	- 数据库
---

![](/images/MySQL.jpg)
>记录安装MySql 过程，并搭建主从模式集群。主从模式在项目中的运用例如读写分离，提高吞吐量在大量需要读操作时可以把压力分散到各个从库不影响主库写操作，如发生主库异常宕机也可以通过从库的数据进行恢复或者顶替主库。
## MySQL 安装
官方网站下载最靠谱，不要在奇奇怪怪的网站下可能会有乱七八糟的插件之类的，唯一指定网站 https://www.mysql.com/ ，本人下载的ZIP包解压，进入文件夹进行数据库配置，默认配置为dafult.ini如果没有则创建`my.ini`配置文件。


## MySQL 同步原理
![](/images/MySqlReplication.jpg)
+ `Master`端需要开启`bin.log`，在每次数据发生改变会往`bin.log`增量写入数据并更新`Pos`，以备下一次增量写入标记`Pos`。
+ `Slave`端的`I/O`读取`master.info`文件，获取`binlog`文件名和位置点并向`Master`端的`I/O`线程发起读取请求。
+ `Master`端的`I/O`线程会根据`Slave`端的`I/O线`程请求信息来读取`binlog`日志信息与及读取到最新的`binlog`文件名和`Pos`一同返回给`Slave`的`I/O`线程。
+ `Slave`端的`I/O`线程会把获取到的`binlog`日志写入`relay`日志（中继日志）文件中，并且更新`master.info`文件信息(包含最后一次读取`Pos`用于下次同步更新的位置点)。
+ `Slave`端的`SQL`线程会定期读取`relay`日志，把二进制的日志解析成`SQL`语句并执行同步数据到从库。

## Master 节点配置
``` ini
[mysql]
# 设置mysql客户端默认字符集UTF8
default-character-set=utf8 
[mysqld]
#设置3306端口
port = 3306 
# 设置mysql的安装目录
basedir=D:\\Program Files (x86)\\mysql-8.0.15-winx64-master
# 设置mysql数据库的数据的存放目录
datadir=D:\\Program Files (x86)\\mysql-8.0.15-winx64-master\\data
# 允许最大连接数
max_connections=200
# 服务端使用的字符集默认为UTF8
character-set-server=utf8
# 创建新表时将使用的默认存储引擎
default-storage-engine=INNODB
# 服务唯一ID
server-id=1
# 开启Log二进制日志
log-bin=master-bin
# 二进制日志记录方式 混合模式
binlog_format=mixed
```

配置完毕进入`bin`文件夹下打开控制台进行安装初始化
``` cmd
# 初始化
mysqld --initialize --console
# 注册服务
mysqld --install [服务名]
# 启动服务
net start [服务名]
```
 __特别注意 `mysqld --initialize --console` 执行会给你初始化密码，使用命令 `mysql -uroot -p` 登录MySQL__

在`master`库中执行以下脚本
``` sql
# 创建用于同步数据的用户
CREATE USER 'slave3307'@'127.0.0.1' IDENTIFIED  BY '123123';
# 赋予权限
GRANT REPLICATION SLAVE,FILE ON *.* TO 'slave3307'@'127.0.0.1';
# 刷新权限
FLUSH PRIVILEGES;
```

## SLAVE 节点配置
``` ini
[mysql]
# 设置mysql客户端默认字符集UTF8
default-character-set=utf8 
[mysqld]
#设置3307端口
port = 3307 
# 设置mysql的安装目录
basedir=D:\\Program Files (x86)\\mysql-8.0.15-winx64-slave
# 设置mysql数据库的数据的存放目录
datadir=D:\\Program Files (x86)\\mysql-8.0.15-winx64-slave\\data
# 允许最大连接数
max_connections=200
# 服务端使用的字符集默认为UTF8
character-set-server=utf8
# 创建新表时将使用的默认存储引擎
default-storage-engine=INNODB
# 服务唯一ID
server-id=2
# 只读设置
read_only=1
# 需要同步的数据库名称，如有多个需配置多条
replicate-do-db=mastersql
```

> `read_only` 这里设置为只读模式，不会影响到`slave`的同步复制功能，可以限制普通用户写入操作防止修改数据导致主从数据不一致，但是无法限制`super`用户的修改数据权限，所以同步复制需要新建一个普通用户用于链接同步。

#### 同步配置
在`5.7`版本之前需要在`my.ini`配置文件`[mysqld]`下添加
``` ini
# 主节点地址
master-host=127.0.0.1
# 主节点端口
master-port=3306
# 主节点复制账号
master-user= slave3307
# 主节点复制密码
master-password= 123123
# 重连时间
master-connect-retry=60
```
`5.7`版本之后的主从配置直接通过动态配置无需修改`ini`
``` sql
# 改变同步配置
change master to master_host='127.0.0.1',master_port=3306, master_user='slave3307', master_password='123123',master_log_file='binlog.000003',master_log_pos=7676;
# 开启同步
start slave;
# 查看同步状态
show slave status;
```
执行`show slave status`语句可以查看当前同步状态，`Slave_IO_Running`和`Slave_SQL_Running`是否为`Yes`，证明同步是否开启成功。`Seconds_Behind_Master`为从库与主库同步位置差异一般为0。执行`stop slave`可停止同步进行修改同步设置然后使用`start slave`重新开启。


##### 语句解释
- `master_host`和`master_port` 为主库地址信息
- `master_port`和`master_password` 同步的账号密码，我们已配置用户为`slave3307`密码为`123`。
- 在`master`库中执行`show master status;`获取`master_log_file` 主库日志和`master_log_pos`当前日志位置，这里可以根据实际情况来设置`master_log_pos`的起始位置。
![主库日志](/images/masterStatus.png)

同步开启后可以尝试在主库下创建一个新数据库`mastersql`,然后新建一张表`mytest`。切换到从库你会发现从库也会自动创建`mastersql`数据库并有一张同名的表`mytest`，说明同步成功。

