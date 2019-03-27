---
title: Linux 关机重启命令
date: 2018-07-25 13:50:04
categories: [Linux]
tags:
    - Linux
---

![Linux](/images/linux-1.jpg)

>记录学习鸟哥的私房菜之开启重启shell笔记，主要有命令shutdown，reboot，halt，poweroff

## 1.Shutdown 命令介绍
- 可以自由的选择关机模式：关机、重启或者进入单用户操作模式即可
- 可以设置关机时间：设置在特定时间或经过多少时长后关闭，也可以立刻关闭
- 可以自定义关机消息：在关闭服务可以通知其他登录的用户
- 可以发送警告命令：在执行一些测试脚本或者可以影响到其他的登录用户的操作时，可以发送警告信息进行提示，但不是真的关机  


``` bash
#脚本参数 shutdown [-t秒] [-arkhncfF] 时间 [警告消息]
-t sec： -t 后单位/秒 经过多少秒后执行
-k      ：不是真关机仅发出警告信息
-r      ：服务关闭后，关闭并重启
-h      : 服务关闭后，立刻关机
-c     ：取消已经在进行中的关闭操作
-f      ：关机启动后，启动略过fsck磁盘检查
-F     ：关机启动后 ，强制进行fsck磁盘检查
# 3600秒后进行关闭并提示警告语
shutdown -t 3600    'Computer will shutdown after 30 min'
```


