---
title: V2Ray 搭建记录
date: 2019-07-17 11:24:48
categories: [技术]
tags:
    - 技术
---

> Vultr 上次的一次特殊事件被封了，后来无意间看到了新的云梯 V2Ray 搭建起来更方便简单，搜了下教程重新搭建并记录

## V2Ray 服务安装
执行官网的一键脚本，执行完毕会自动安装好`unzip`与`deamon`软件，应用程序默认安装在`/usr/bin/v2ray/v2ray`，当有新版本发布只需再执行该脚本即可

``` bash
bash <(curl -L -s https://install.direct/go.sh)
```

#### V2Ray 配置文件
安装完毕可以通过`cat /etc/v2ray/config.json`查看配置文件，可以通过`vi`命令修改保存配置文件达到自定义
``` json
{
  "inbounds": [{
    "port": 10067,
    "protocol": "vmess",
    "settings": {
      "clients": [
        {
          "id": "08e9a92c-8f5d-45ce-8791-44f0d13379e7",
          "level": 1,
          "alterId": 64
        }
      ]
    }
  }],
  "outbounds": [{
    "protocol": "freedom",
    "settings": {}
  },{
    "protocol": "blackhole",
    "settings": {},
    "tag": "blocked"
  }],
  "routing": {
    "rules": [
      {
        "type": "field",
        "ip": ["geoip:private"],
        "outboundTag": "blocked"
      }
    ]
  }
}
```

- 配置文件中的id、端口、alterId需要和客户端的配置保持一致
- 服务端使用脚本安装成功之后默认就是`vmess`协议

#### V2Ray 相关脚本
第一次安装完毕`V2Ray`不会自行启动需要通过以下脚本启动、停止、查看状态
``` bash
## 启动
systemctl start v2ray

## 停止
systemctl stop v2ray

## 状态
systemctl status v2ray
```
由于防火墙原因会导致启用了服务但是无法连接问题，需要把服务相关端口加入防火墙白名单
``` bash
## 查看开启的端口
firewall-cmd --zone=public --list-ports

## 加入新端口 100067 80 端口
firewall-cmd --zone=public --add-port=80/tcp --permanent
firewall-cmd --zone=public --add-port=10067/tcp --permanent
```

## V2RayN 客户端
下载 [V2Ray-Core](https://github.com/v2ray/v2ray-core/releases) 解压点击`v2rayN.exe`应用添加`Vmess`服务器
![](/images/vmess-windows-client.jpg)

填写完毕启用`Http`代理修改代理模式为`PAC`模式，然后就可以愉快的在上google

## BBR 加速
搭建好了`V2Ray`你感觉速度不够快连接效果不够好，我们可以通过脚本开启`BBR`
``` bash
#安装wget，digitalocean默认没有安装wget，安装一下
yum -y install wget

#执行BBR PLUS修正版一键脚本
wget -N --no-check-certificate "https://raw.githubusercontent.com/chiakge/Linux-NetSpeed/master/tcp.sh" && chmod +x tcp.sh && ./tcp.sh
```
根据自身情况安装内核重启系统后安装`BBR加速`