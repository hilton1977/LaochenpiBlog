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
``` bash
user nobody nobody;
worker_processes 2;
error_log logs/error.log notice;
pid logs/nginx.pid;
worker_rlimit_nofile 65535;
 
events{
use epoll;
worker_connections 65536;
}
```

- user 指定 Nginx Woker 进程运行用户以及用户组，默认值 nobby nobby
- worker_processes 配置工作进程数，建议于CPU核心数量一致，auto为自动检测
- error_log 日志文件的地址和日志级别，debug、info、notice、warn、error、crit
- pid pid进程标识符存放地址
- worker_rlimit_nofile worker进程可以打开最大文件限制数


### Events 模块
影响nginx服务器与用户的网络连接，通常有是否开启对WP下的网络进行序列化 是否允许同时接受多个网络连接 事件驱动模型 每个WP可以同时支持处理的最大连接数

``` bash
events {
    worker_connections 2048;
    multi_accept on;
    use epoll;
    keepalive_timeout 60;
}
```

- worker_connections 每个进程的最大连接数，客户端连接数 MaxClient = worker_processes * worker_connections，Linux 系统下进程最大连接数受最大文件句柄数所限制，通过`ulimit -n 连接数`来改变其最大打开文件数
- use 事件驱动模式： kqueue | rtsig | epoll | /dev/poll | select | poll | eventport
- multi_accept 是否允许同时接受多个连接
- keepalive_timeout 连接超时时间单位（秒）

### Http 模块
server（主机设置）、upstream（负载均衡服务器设置）和 location（URL匹配特定位置的设置）

```
http {
    server_tokens off;
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;

    ...
}
```