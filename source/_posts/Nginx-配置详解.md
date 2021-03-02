---
title: Nginx 配置详解
tags:
  - 技术
categories:
  - 技术
toc: false
date: 2019-08-09 16:33:35
---

![](/images/nginxconf.png)

> 学习 Nginx的配置文件用于以后回来查询回顾

### Core 模块
- 配置运行Nginx服务器用户（组）
- worker process数
- Nginx进程PID存放路径
- 错误日志的存放路径
- 配置文件的引入
``` bash
user nobody nobody;
worker_processes 2;
error_log logs/error.log notice;
pid logs/nginx.pid;
worker_rlimit_nofile 65535;
```
##### 基本参数

|参数|描述|
|-|-|
|user|指定 Nginx Woker 进程运行用户以及用户组，默认值 nobby nobby|
|worker_processes|配置工作进程数，建议于CPU核心数量一致，auto为自动检测|
|error_log|日志文件的地址和日志级别，debug、info、notice、warn、error、crit|
|pid|pid进程标识符存放地址|
|worker_rlimit_nofile worker|进程可以打开最大文件限制数|


### Events 模块
- 该部分配置主要影响Nginx服务器与用户的网络连接，主要包括：
- 设置网络连接的序列化
- 是否允许同时接收多个网络连接
- 事件驱动模型的选择
- 进程连接数的配置

``` bash
events {
    worker_connections 2048;
    multi_accept on;
    use epoll;
    keepalive_timeout 60;
    accept_mutex on;
}
```
|参数|描述|
|-|-|
|worker_connections |每个进程的最大连接数，客户端连接数 MaxClient = worker_processes * worker_connections，Linux 系统下进程最大连接数受最大文件句柄数所限制，通过`ulimit -n 连接数`来改变其最大打开文件数|
|use |事件驱动模式 kqueue、rtsig、epoll、/dev/poll、select、poll 、eventport|
|multi_accept|是否允许同时接受多个连接|
|keepalive_timeout | 连接超时时间单位（秒）|
|accept_mutex |网路连接序列化，防止__惊群现象__发生，默认为on|

##### 惊群现象
![](/images/thundering-herd.png)
当`CreateSocket`发生连接事件时，所有的`fork`进程都被唤醒，而最终只有一个进程能处理事件响应，其余进程无法获取事件则只能重新睡眠或其他操作，这样现象我们称之为__惊群现象__

### Http 模块
- 定义MIMI-Type
- 自定义服务日志
- 允许sendfile方式传输文件
- 连接超时时间
- 单连接请求数上限
``` bash
http {
    include mine.types;
    server_tokens off;
    default_type application/octet-stream
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65；
    ...
}
```
##### 基本参数

|参数|描述|
|-|-|
|include|将其他的Nginx配置或者第三方模块的配置引用到当前的主配置文件中|
|keepalive_timeout|客户端连接保持活动的超时时间|
|sendfile on |开启高效文件传输模式|
|default_type |默认类型为二进制流|

#### Upstream 负载均衡模块
负载均衡目前支持4种方法：

|方法|描述|
|-|-|
|poll 轮询|默认方法按照请求时间依次分发到配置的服务|
|weight 权重|根据服务配置的权重来进行分配请求|
|ip_hash|根据请求 ip 的 hash 值来固定访问服务，解决了服务 session 不一致问题|
|fair|根据请求的响应时间智能分配服务|
|url_hash|根据请求的 url 的 hash 地址分配服务|

##### 基本参数

|参数|描述|
|-|-|
|fail_timeout|失败时间周期|
|max_fails|在 fail_timeout 周期内之内所有请求（最大失败次数）均失败则认为服务停机，等待 fail_timeout 周期后再请求|
|backup	|服务标记为热备状态，当其他服务为停机状态下请求才会被分配热备服务|
|down|服务被标记为停机状态|
|max_conns|最大连接数|

``` bash
upstream mysvr {   
      server 127.0.0.1:7878;
      server 192.168.10.121:3333 backup;  #热备
}
```

#### Server 配置
- 配置网络监听
- 基于名称的虚拟主机配置
- 基于IP的虚拟主机配置

``` bash
server {
        keepalive_requests 120; #单连接请求上限次数。
        listen       4545;   #监听端口
	    acccess_log /var/access.log;
        server_name  127.0.0.1;   #监听地址      
	    root /var/www;

        location  ~*^.+$ {      
           proxy_pass  http://mysvr;
           deny 127.0.0.1;  #拒绝的ip
           allow 172.18.5.54; #允许的ip           
        } 
}
```
##### 基本参数
|参数|描述|
|-|-|
|listen|监听配置格式`[ip:port]`、`[ip]`、`[port]`|
|server_name|监听域名可用正则进行通配|
|access_log|访问记录日志地址|
|keepalive_requests|单连接请求上线次数|
|root|作为根目录用于文件的检索|

##### Location 配置
通过指定模式来与客户端请求的URI相匹配，基本语法如下：`location [=|~|~*|^~|@] pattern{……}`

|配置|匹配|
|-|-|
|/|模糊匹配|
|=|精确匹配|
|^~|标示 uri 以指定字符串开头|
|~|正则匹配需要区别大小写|
|~*|正则匹配不需要区别大小写|
|@|指定区段无法访问|

##### 优先级： = 高于 ^~ 高 ~* 等于 ~ 高于 /

##### 基本参数

|参数|描述|
|-|-|
|proxy_pass|代理转发 proxy_pass后面的url加`/`，表示`绝对根路径`；如果没有`/`，表示`相对路径`|
|alias|访问文件路径别名|

###### 例子
``` bash
location /mytest/ {
	proxy_pass http://127.0.0.1:8080/abc;
}

# 访问 http://127.0.0.1/mytest/test1 代理转发 http://127.0.0.1:8080/abctest1

location /mytest/{
	proxy_pass http://127.0.0.1/8080/abc/;
}

#访问 http://127.0.0.1/mytest/test1 代理转发 http://127.0.0.1:8080/abc/test1
```

> 推荐使用 https://nginxconfig.io/ 可自定义配置