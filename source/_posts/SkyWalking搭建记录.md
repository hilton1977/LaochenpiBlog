---
title: SkyWalking 搭建记录
date: 2019-04-02 11:08:24
categories: [Java]
tags:
	 - 中间件 
---

![](/images/SkyWalking.jpg)

> 记录搭建Skywalking 6.0过程和详细遇到的问题

## SkyWalking 
`SkyWalking` 是观察性分析平台和应用性能管理系统，提供分布式追踪、服务网格遥测分析、度量聚合和可视化一体化解决方案。它是一款国产开源软件源码在github上，
官方网站为 http://skywalking.apache.org/zh/ Github https://github.com/apache/incubator-skywalking/

![项目结构](/images/skywalkingFile.png)
### Agent 
`Agent`探针项目里面包含`skywalking-agent.jar`，使用`JavaAgent`做字节码侵入，对代码无侵入，收集格式化数据然后通过`Http`或`gRpc`方式发送到`SkyWalking Collector`。
通过`agent.config`配置探针
- __agent.namespace__： 跨进程链路中的`header`，不同的`namespace`会导致跨进程的链路中断
- __agent.service_name__：一个服务（项目）的唯一标识，这个字段决定了在sw的UI上的关于`service`的展示名称
- __agent.sample_n_per_3_secs__：客户端采样率，默认是-1代表全采样
- __agent.authentication__： 与`collector`进行通信的安全认证，需要同`collector`中配置相同
- __agent.ignore_suffix__：忽略特定请求后缀的trace
- __collector.backend_service__：`agent`需要同`collector`进行数据传输的IP和端口
- __logging.level__：`agent`记录日志级别

##### 集成方式
常用的`Java`项目运行方式`Jar`方式或者`War`通过容器启动，例如`Tomcat`
- __Jar__ 配置JVM运行添加探针 `-javaagent:[skywalking-agent.jar绝对路径]`
- __Tomcat__ `tomcat`目录`bin`下的`catalina`脚本
``` shell
Window
CATALINA_OPTS = -javaagent:[skywalking-agent.jar绝对路径]
Linux
CATALINA_OPTS="$CATALINA_OPTS -javaagent:[skywalking-agent.jar绝对路径]"; export CATALINA_OPTS
```

### Config
`Config`文件夹中包含了`application.yml`配置文件，可以配置收集的数据存储方式，默认是`h2`一般使用`elasticsearch`作为存储方式。
``` yml
storage:
 elasticsearch:
   nameSpace: es5-cluster
   clusterNodes: 192.168.105.13:9200
   indexShardsNumber: 2
   indexReplicasNumber: 0
   # Batch process setting, refer to https://www.elastic.co/guide/en/elasticsearch/client/java-api/5.5/java-docs-bulk-processor.html
   bulkActions: 2000 # Execute the bulk every 2000 requests
   bulkSize: 20 # flush the bulk every 20mb
   flushInterval: 10 # flush the bulk every 10 seconds whatever the number of requests
   concurrentRequests: 2 # the number of concurrent requests
```

启动`bin`文件夹中的`startup.bat`或者`start.sh`根据系统来执行，会启动2个项目`Skywalking-Collector`用于收集数据并把数据存储到Es中，`Skywalking-WebApp`用于展示收集的数据，2个项目的日志在`logs`中记录生成，通过http://localhot:8080查看运行情况

![Skywalking-WebApp](/images/skywalking-ui.png)

## 注意事项
- `Skywalking` __6.0__ 相对 __5.0__简化了配置项，数据落地添加了`MySql`方式，需要注意`Elasticsearch`要求的版本也不一样，从`5.X`到`6.X`
- `Skywalking UI`默认的登录密码为`admin`，可以在`webapp.yml`中自行配置
- `Skywalking-WebApp`和`Skywalking-Collector`如果跟探针不在同一机器上，修改`Collector`的配置文件`application.yml`的`host`，以便于探针收集的数据能准确的发送到`Collector`，同时修改探针`agent.config`配置项`collector.servers`地址。
``` yml
core:
  default:
    restHost: 192.168.104.162
    restPort: 12800
    restContextPath: /
    gRPCHost: 192.168.104.162
    gRPCPort: 11800
    downsampling:
    - Hour
    - Day
    - Month
    # Set a timeout on metric data. After the timeout has expired, the metric data will automatically be deleted.
    recordDataTTL: 90 # Unit is minute
    minuteMetricsDataTTL: 90 # Unit is minute
    hourMetricsDataTTL: 36 # Unit is hour
    dayMetricsDataTTL: 45 # Unit is day
    monthMetricsDataTTL: 18 # Unit is month
```
- `naming`中的地址对应了`webapp.yml`中`listOfServers: 127.0.0.1:12800`，`UI`使用`rest http`通信对应配置文件的`restHost`和`restPort`，`agent`在大多数场景下使用`gRpc`方式通信，在语言不支持的情况下会使用`http`通信。

##### 参考资料 http://www.primeton.com/read.php?id=2751