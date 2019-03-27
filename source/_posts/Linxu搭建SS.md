---
title: Linux 搭建 Shadowsocks
date: 2018-07-13 16:25:43
categories: [Linux]
tags:
    - Linux
---

![Shadowsocks](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1531801271142&di=761db1c2eaadf98b71507ffb63bdde32&imgtype=0&src=http%3A%2F%2Fn1.itc.cn%2Fimg8%2Fwb%2Fsmccloud%2Frecom%2F2015%2F07%2F14%2F143686120068151458.JPEG)

>作为一个码农没有科学上网怎么能行，刚好Vultr新注册送钱买一个云主机玩玩，
以CentOs7做一个教程，之前在网上找的搭建方法很多错误导致一直不成功现在自己整理并通过测试，踩了很多坑


## 1.Shadowsocks 环境准备
 
``` bash
#安装epel扩展源
yum install epel-release
#安装Pip
yum -y install python-pip
#升级Pip
pip install --upgrade pip 
#清除yum缓存
yum clean all
#安装shadowsocks客户端
pip install shadowsocks
```

## 2.Shadowsocks 配置

``` bash
#创建shadowsocks配置
vi /etc/shadowsocks.json
#单用户
 { 
    "server":"server_ip"， 
    "server_port":25， 
    "local_address": "127.0.0.1"， 
    "local_port":1080， 
    "password":"password"，
    "timeout":300， 
    "method":"aes-256-cfb"， 
    "fast_open": false 
 }

#多用户
{
    "server":"server_ip"，
    "port_password":{
        "port_1":"pwd1"，
        "port_2":"pwd2"，
        "port_3":"pwd3"
    }，
    "local_address":"127.0.0.1"，
    "local_port":1080，
    "timeout":300，
    "method":"aes-256-cfb"
}
```
**参数详解**:
* server 服务器地址 127.0.0.1 或者0.0.0.0
* server_port 服务端口号 外部连接需要填写的服务端口号
* local_port 本地端口号
* password 连接密码
* timeout 超时时间
* method 加密方式

## 3.Shadowsocks 启动
``` bash   
#启动
ssserver -c /etc/shadowsocks.json -d start
#停止
ssserver -c /etc/shadowsocks.json -d stop
```
由于每次都需要服务器重启都需要手动去启动不便，可以注册成服务自动启动
``` bash
#创建服务脚本 servicename 填写shadowsocks
vi /etc/systemd/system/[servicename].service

#编辑脚本
[Unit]
Description=Shadowsocks

[Service]
TimeoutStartSec=0
ExecStart=/usr/bin/ssserver -c /etc/shadowsocks.json start
ExecStop=/usr/bin/ssserver -c /etc/shadowsocks.json stop


[Install]
WantedBy=multi-user.target
```
这里会遇到一个坑：ExecStart 这里填写的启动脚本 少了一个start不知道是不是我本身脚本问题

**参数详解**:
* Description服务描述
* ExecStart 服务启动执行脚本
* ExecStop 服务停止执行脚本
* WantedBy 系统以该形式运行时，服务方可启动

## 4.Systemctl 命令
注册服务 `systemctl enable shadowsocks`
所有服务 `systemctl list-units --type=service`
服务状态 `systemctl status shadowsocks -l `
启动服务 `systemctl start shadowsocks`
停止服务 `systemctl stop shadowsocks`
重启服务 `systemctl restart shadowsocks`

## 5.Shadowsocks 客户端安装

环境支持
- [Shadowsocks for Win](https://github.com/shadowsocks/shadowsocks-windows/releases)
- [Microsoft .NET Framework 4.6.2](https://www.microsoft.com/en-US/download/details.aspx?id=53344) 
- [Microsoft Visual C++ 2015 Redistributable (x86) ](https://www.microsoft.com/en-us/download/details.aspx?id=53840)
安装完毕配置启动即可

### 贴士提示
* CentOs7需要配置下防火墙端口白名单
``` bash
#添加端口号8388(设置的server-port) --permanent永久生效
firewall-cmd --zone=public --add-port=8388/tcp --permanent 
#重载配置
firewall-cmd --reload
```
