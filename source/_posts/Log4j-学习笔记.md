---
title: Log4j 自定义多文件分离
date: 2018-07-20 17:32:59
categories: [Spring]
tags:
    - Java
---
![Log4j](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1533619453&di=71e7053f6a5104d2ee0501827d562550&imgtype=jpg&er=1&src=http%3A%2F%2Fstatic.open-open.com%2Fnews%2FuploadImg%2F20160530%2F20160530232653_508.jpg)

>在工作开发中遇到一个需求需要通过某一些条件逻辑进行分组细化日志，用配置的一些条件进行不同的日志管理和处理，由于之前的日志没有细化会导致在很多日志中无法更快和更精准的定位某一个模块的错误，如大海捞针效率极低，细分后方便开发和维护人员对日志更快更精准的排查修改BUG。

## 1.Log4j 介绍
   Log4j有三个主要的组件：Loggers(记录器)，Appenders (输出源)和Layouts(布局)。这里可简单理解为日志类别，日志要输出的地方和日志以何种形式输出。综合使用这三个组件可以轻松地记录信息的类型和级别，并可以在运行时控制日志输出的样式和位置。
             
## 2.Log4j 组件
### Appender 配置
 - ##### ConsoleAppender (控制台)
    1. Threshold=WARN：指定日志信息的最低输出级别，默认为DEBUG。
    2. ImmediateFlush=true：消息都会被立即输出，设为false则不输出，默认值是true。
    3. Target=System.err：默认值是System.out。
 - ##### FileAppender (文件)
    1. Append=false：true表示消息增加到指定文件中，false则将消息覆盖指定的文件内容，默认值是true。
    2. File=D:/logs/logging.log4j：指定消息输出到logging.log4j文件中。
 - ##### DailyRollingFileAppender (按照日期格式生成)
    1. DatePattern='.'yyyy-MM：根据时间格式按照年月日为单位生成log文件
     '.'yyyy-MM：每月
     '.'yyyy-ww：每周
     '.'yyyy-MM-dd：每天
     '.'yyyy-MM-dd-a：每天两次
     '.'yyyy-MM-dd-HH：每小时
     '.'yyyy-MM-dd-HH-mm：每分钟
 - ##### RollingFileAppender (文件大小到达指定尺寸的时候产生一个新的文件)
    1. MaxFileSize=100KB：后缀可以是KB， MB 或者GB。在日志文件到达该大小时，将会自动滚动，即将原来的内容移到logging.log4j.1文件中。
    2. MaxBackupIndex=2：指定可以产生的滚动文件的最大数，例如，设为2则可以产生logging.log4j.1，logging.log4j.2两个滚动文件和一个logging.log4j文
 - ##### SocketAppender (发送远程服务 Tip:可配合logstash使用)
    1. host，String，指定服务器的主机名。（必需）
    2. immediateFlush，boolean，是否立即flush，还是等待缓存到一定大小后在flush。
    3. layout，Layout，log event输出的格式。
    4. port，integer，远程服务器坚挺log event的应用的端口号。
    5. protocol，String，发送log event所使用的协议，"TCP" 或"UDP"。
    6. reconnectionDelay，integer，当连接断开时，延迟等待的ms数。
    7. name，String ，Appender的名称。
    8. protocol，String，通讯协议 默认TCP。可选值 "TCP" (default)， "SSL" or "UDP".
    9. SSL，SslConfiguration，包含密钥存储库和信任存储库的配置.
    10. filter，Filter，一个过滤器来确定事件应该由这个Appender。 不止一个过滤器 可以通过使用一个CompositeFilter。
    11. immediateFail，boolean，设置为true时，日志事件不会等待尝试重新连接，将立即如果失败 套接字是不可用的。
    12. immediateFlush，boolean， 当该值设置成真时，默认情况下，每个写将冲洗。 这将保证写的数据 到磁盘，但可能会影响性能。
    13. layout，Layout，LogEvent ，布局使用格式。 缺省值是SerializedLayout。
    14. reconnectionDelay，integer ，如果设置为值大于0，一个错误后SocketManager将尝试重新连接 在指定的毫秒数后的服务器。 如果连接失败 将抛出一个异常(可以被应用程序如果ignoreExceptions是 设置为假)。
    15. ignoreExceptions，boolean，默认值是真正的添加事件时，遇到了引起异常 内部记录，然后忽略。 当设置为假将传播到异常 调用者。 你必须设置这个假当包装这个AppenderFailoverAppender。
 - ##### SMTPAppender (发送邮件)
    1. smtpHost= mtp.163.com：邮件服务器地址
    2. smtpPort=30 ：端口号
    3. from= ***@**.com：发送方邮箱
    4. replyTo = ***@**.com： 接收方方邮箱
    5. smtpUsername = 285635652@qq.com：发送方邮箱账号
    6. smtpPassword = **********：发送方邮箱密码

    
>log4j.additivity.[appenderName]=false (用于独立输出日志，Logger只会在自己的appender里输出，而不会在父Logger的appender里输出。)默认为true


### Layouts
- ##### HTMLLayout（以HTML表格形式布局） 
- ##### PatternLayout（可以灵活地指定布局模式） 
- ##### SimpleLayout（包含日志信息的级别和信息字符串） 
- ##### TTCCLayout（包含日志产生的时间、线程、类别等信息）   

## 3.Spring 运用 Log4j
``` properties
# LOG4J配置
log4j.rootCategory=INFO， stdout， file
log4j.logger.errorfile=error，errorfile

# 控制台输出
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss，SSS} %5p %c{1}:%L - %m%n

# root日志输出
log4j.appender.file=org.apache.log4j.DailyRollingFileAppender
log4j.appender.file.file=logs/all.log
log4j.appender.file.DatePattern='.'yyyy-MM-dd
log4j.appender.file.layout=org.apache.log4j.PatternLayout
log4j.appender.file.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss，SSS} %5p %c{1}:%L - %m%n

# error日志输出
log4j.appender.errorfile=org.apache.log4j.DailyRollingFileAppender
log4j.appender.errorfile.file=logs/error.log
log4j.appender.errorfile.DatePattern='.'yyyy-MM-dd
log4j.appender.errorfile.Threshold = ERROR
log4j.appender.errorfile.layout=org.apache.log4j.PatternLayout
log4j.appender.errorfile.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss，SSS} %5p %c{1}:%L - %m%n

#自定义业务分组 team mytest输出目标
log4j.logger.team=INFO，mytest
#自定义日志输出
#输出的各种Appender
log4j.appender.mytest=org.apache.log4j.DailyRollingFileAppender
#父类节点不输出 分级
log4j.additivity.team=false
#输出的日志地址
log4j.appender.mytest.file=logs/mytest.log
#记录的时间单位 天 
log4j.appender.mytest.DatePattern='.'yyyy-MM-dd
#布局
log4j.appender.mytest.layout=org.apache.log4j.PatternLayout
#输出内容
log4j.appender.mytest.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss，SSS} %5p %c{1}:%L ---- %m%n

```

#### 讲解
1. rootCategory 主节点 [日志级别]，[输出目标]，[输出目标]，[...]
2. category 子节点 特别会集成主节点的设置 日志级别
3. log4j.appender.[输出目标] 日志的输出设置 包含输出格式、布局、方式等
4. 优先级：DEBUG < INFO < WARN < ERROR < FATAL
5. PatternLayout 布局 ConversionPattern相关设置  
%m 输出代码中指定的消息
%p 输出优先级，即DEBUG，INFO，WARN，ERROR，FATAL
%r 输出自应用启动到输出该log信息耗费的毫秒数
%c 输出所属的类目，通常就是所在类的全名
%t 输出产生该日志事件的线程名
%n 输出一个回车换行符，Windows平台为“rn”，Unix平台为“n”
%d 输出日志时间点的日期或时间，默认格式为ISO8601，也可以在其后指定格式，比如：%d{yyyy MMM ddHH:mm:ss，SSS}，输出类似：2002年10月18日 22：10：28，921
%l 输出日志事件的发生位置，包括类目名、发生的线程，以及在代码中的行数。
[QC]是log信息的开头，可以为任意字符，一般为项目简称。
