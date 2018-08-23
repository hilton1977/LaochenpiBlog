---
title: Logstash同步数据库
date: 2018-05-25 16:27:08
tags:
    - logstash,ELK
    - 技术
---
![](/images/es.jpg)

>由于业务需求需要同步某些数据库的表数据更新修改删除需同步ES保证同步性，
在进行curd用AOP可实现同步，但是考虑到解耦分离后续系统水平拓展，
查询资料可以用Logstash进行同步Es,Logstash 是开源的服务器端数据处理管道，
能够同时 从多个来源采集数据、转换数据，然后将数据发送到Elasticsearch.



## 1.Logstash依赖环境
* JDK1.8 [下载地址](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)
* Ruby环境 [下载地址](http://www.ruby-lang.org/en/downloads/)
* logstash 6.3.1 [下载地址](https://www.elastic.co/downloads/logstash)

## 2.Logstash同步配置文件
>Logstash由三个组件构造成，分别是input、filter以及output。我们可以吧Logstash三个组件的工作流理解为：input收集数据，filter处理数据，output输出数据。至于怎么收集、去哪收集、怎么处理、处理什么、怎么发生以及发送到哪等等一些列的问题就是我们接下啦要讨论的一个重点。
```
input {
    jdbc{
    #数据库驱动jar包
        jdbc_driver_library => "\policySyn\ojdbc6.jar"
    #数据库地址
        jdbc_connection_string => "jdbc:oracle:thin:@192.168.105.16:1523:gnnt"
    #数据库用户名密码
        jdbc_user => "plane_tick"
        jdbc_password => "ora123"
    #数据库驱动类
        jdbc_driver_class => "Java::oracle.jdbc.driver.OracleDriver"
        jdbc_paging_enabled => "true"
        jdbc_page_size => "50000"
    #执行sql绝对路径 或相对路径
        statement_filepath  => "\policySyn\syn.sql"
    #更新时间记录和存放
        record_last_run => "true"
        last_run_metadata_path => "\policySyn\synDate.txt"
    #定时更新频率 20分钟一次
        schedule => "* * * * *"
    #索引类型
        type => "policyteam_dev"
    }
}

//同步目的地
output {
    elasticsearch{
        hosts => "http://192.168.105.13:9200"
        index => "policyteam_dev"
        document_id => "%{zcbh}"
    }
    stdout {
        codec => json_lines
    }
}
```
## 3.启动同步脚本
进入Logstash目录bin文件夹下执行脚本
``` bash 
    #config为执行配置文件绝对路径或相对路径
    logstash -f [config]
```







