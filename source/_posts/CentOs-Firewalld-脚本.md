---
title: CentOs FireWall 脚本
date: 2018-07-19 14:20:43
categories: [Linux]
tags:
    - Linux
---

![Firewalld](http://www.tecmint.com/wp-content/uploads/2016/01/Enable-Disable-Iptables-FirewallD.png)

>经过之前自己搭建了Shadowsocks接触Linux慢慢想深入学习下一些常用Shell，之前在配置Shadowsocks遇到启动服务但是PC客户端连接没有网络，通过查阅一些教程发现Centos7默认开启了防火墙Firewall导致如果没有开放Shadowsocks的相关端口是无法访问的，现在记录下Firewall的一些相关命令

##  1.Firewalld 简介
CentOs7的一大特性，最大的好处有两个：支持动态更新，不用重启服务；第二个就是加入了防火墙的“zone”概念，有图形界面和工具界面，由于我在服务器上使用，图形界面请参照官方文档，本文以字符界面做介绍，firewalld的字符界面管理工具是 firewall-cmd 默认配置文件有两个：/usr/lib/firewalld/ （用户配置地址） 和 /etc/firewalld/ （系统配置，尽量不要修改）

## 2.Zone 概念
Firewall 能将不同的网络连接归类到不同的信任级别，Zone 提供了以下几个级别
* drop: 丢弃所有进入的包，而不给出任何响应
* block: 拒绝所有外部发起的连接，允许内部发起的连接
* public: 允许指定的进入连接
* external: 同上，对伪装的进入连接，一般用于路由转发
* dmz: 允许受限制的进入连接
* work: 允许受信任的计算机被限制的进入连接，类似 workgroup
* home: 同上，类似 homegroup
* internal: 同上，范围针对所有互联网用户
* trusted: 信任所有连接

## 3.过滤规则
过滤规则的优先级遵循如下顺序source>interface>firewalld.conf
* source: 根据源地址过滤
* interface: 根据网卡过滤
* service: 根据服务名过滤
* port: 根据端口过滤
* icmp-block: icmp 报文过滤，按照 icmp 类型配置
* masquerade: ip 地址伪装
* forward-port: 端口转发
* rule: 自定义规则

## 4.使用方法
firewall-cmd [指令] 
--zone 作用域  
--permanent  永久修改  
--reload 重载生效 
--timeout=seconds 持续时间，一般用于调试            
          
__使用实例__:
``` bash
#查看开放的Zone
firewall-cmd --get-active-zones
#查看firewalld状态
firewall-cmd --state
#查看firewalld开放的端口
firewall-cmd --zone=dmz --list-ports
#重新加载配置 (无需重启)
firewall-cmd --reload
#重新加载配置 (重启服务器加载)
firewall-cmd --complete-reload 
#添加一个端口允许访问 (临时添加)
firewall-cmd --zone=dwz --add-port=8080/tcp
#添加一个端口允许访问 (永久添加)
firewall-cmd --zone=dwz --add-port=8080/tcp --permanent
#添加一个端口允许访问 (持续300秒)
firewall-cmd --zone=dwz --add-port=8080/tcp --timeout=300
#添加一个服务允许访问
firewall-cmd --zone=dwz --add-service=smtp
#启用firewalld
systemctl start firewalld
#停止firewalld
systemctl stop firewalld
#重启firewalld
systemctrl restart firewalld
#禁用firewalld
systemctrl disable firewalld
```
