---
title: Winsw把java项目做成服务
date: 2018-08-16 20:04:17
tags: 
	- Java
	- 心得
---

>jar项目需要通过命令行jar -jar 执行脚本启动显示控制台,由于强迫症可以使用javaw -jar来执行可以在后台执行，但通过java编译启动在window环境下进程名都为java.exe一旦项目多了当你要更新部署更新关闭项目时候就懵逼了有可能就会误操作，通过Google发现有个开源的软件
winsw 可以把任何软件做为window 的服务来管理，这样在services.msc 服务管理里可以很方便的进行管理更新部署。

## 1.Winsw  环境
Winsw是个开源项目,Github地址为:https://github.com/kohsuke/winsw 依赖环境为NET2 或 NET4, 可通过配置文件进行修改。

## 2.JAVA项目注册服务
根据作者的介绍注册的服务依赖于配置文件 *.xml，这里需要注意的是xml的文件名称必须和winsw.exe同名。默认是按软件的名称来匹配配置文件。例如你把winsw.exe重复名为test.exe那配置文件必须为test.xml不然不无法使用。
``` xml
<service>
  <id>MyTest</id>
  <name>MyTest</name>
  <description>测试jar项目服务</description>
  <env name="JENKINS_HOME" value="%BASE%"/>
  <executable>java</executable>
  <arguments>-Xrs -Xmx256m -jar "%BASE%\test.jar" --httpPort=8080</arguments>
  <logmode>rotate</logmode>
</service>
```
配置文件解释:
- id：服务名称 (唯一)
- name：显示服务名称
- description：服务描述
- env：环境变量 JENKINS_HOME 赋值给 %BASE%
- executable：执行命令 这里我们是用java启动
- arguments：执行的一些参数
- logmode：日志模式
这里 executable arguments 就相当于你在控制台执行的脚本，根据你的需求进行改变命令和参数。
通过控制台进入winsw软件目录执行`` winsw.exe install`` 注册服务, winsw为软件名称可以自行修改。执行成功可以在控制看到
![注册成功](/images/winsw.png)

如果发现错误请查看 `[软件名称].wrapper.log` 日志排查，是否配置文件名和软件名不一致或者配置的地址不存在等。然后你可以通过 services.msc 对你的服务进行操作了启动，停止。注册的服务默认是AutoStart每次重启电脑都会自动启动。

配置文件的相关其他设置可以参考: https://github.com/kohsuke/winsw/blob/master/doc/xmlConfigFile.md
