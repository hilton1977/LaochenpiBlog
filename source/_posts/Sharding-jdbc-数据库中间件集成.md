---
title: Sharding-jdbc 数据库中间件集成
tags:
  - 中间件
categories:
  - 技术
toc: false
date: 2019-10-17 16:42:30
---

> 在开发政策推送项目发现我们的控润表越来越大将近千万级查询效率极低，历史垃圾数据删除也比较麻烦后期还需要新增各种航司数据量会更大，才想起来可以将控润按月份进行分表操作，历史数据直接 truncate 查询效率变高，这里使用了sharding-jdbc进行自动分表


### Sharding-JDBC
根据官方的介绍 Sharding-JDBC 属于一款轻量级的 Java 框架，直接通过引入相应的 jar 包和设置配置项达到分库分表、读写分离等功能不需额外的服务支撑，对于现有项目代码零侵入，兼容大多数 Orm 框架 （Hibernate、Spring-JPA、Mybatis）和各类第三方 JDBC 数据库连接池（Druid、DBCP、HikariCP）。

![](/images/sharding-jdbc.png)

> 官方的文档: https://shardingsphere.apache.org/document/current/cn/overview/

### SpringBoot 集成
maven 项目通过引入`sharding-jdbc-spring-boot-starter`进行配置
``` xml
<dependency>
  <groupId>org.apache.shardingsphere</groupId>
  <artifactId>sharding-jdbc-spring-boot-starter</artifactId>
  <version>4.0.0-RC1</version>
</dependency>
```

Sharding-JDBC 的配置有3种根据项目情况自行选择，这里采用 `yml`方式集成开发，需要修改原有的`datasource`替换成`sharding.datasource`，配置如下

``` yml
spring:
  profiles: dev
  shardingsphere:
    datasource: # 数据库相关设置
      names: ds
      ds:
        type: com.alibaba.druid.pool.DruidDataSource
        driver-class-name: oracle.jdbc.OracleDriver
        url: jdbc:oracle:thin:@192.168.105.16:1523:TKPO
        username: ticket_policy
        password: ticket_policy
    sharding: # 分片设置
      tables:
        filter_control_setting: # 逻辑表
          actual-data-nodes: ds.filter_control_setting_$->{1..12} # 实际物理表集合
          table-strategy:
            standard:
              sharding-column: departdate # 分片字段
              range-algorithm-class-name: com.zkxy.data.config.FilterControlRangeShardingAlgorithm
              precise-algorithm-class-name: com.zkxy.data.config.FilterControlPreciseShardingAlgorithm
    props:
      sql.show: true
```

##### 配置解释
- `datasource` 数据库相关信息
- `sql.show` 打印出sql执行情况
- `filter_control_setting` 逻辑表名，在执行`sql`该逻辑表时会根据分词进行转化成真正的物理表名
- `actual-data-nodes` 真实的物理表集合，`$->{1..12}`这里指1-12后缀，12张平行表也叫水平分表
- `sharding-cloumns` 分片列，对应数据库字段多个用逗号隔开
- `algorithm-class-name` 分片逻辑实现类路径
- `table-strategy` 分表算法逻辑，官方提供了4种分片算法:
	1. 精确分片算法 `PreciseShardingAlgorithm` 使用单一键作为分片键的=与IN进行分片的场景
	
	2. 范围分片算法 `RangeShardingAlgorithm` 使用单一键作为分片键的BETWEEN AND进行分片的场景
	
	3. 复合分片算法 `ComplexKeysShardingAlgorithm` 使用多键作为分片键进行分片的场景，包含多个分片键的逻辑较复杂，需要应用开发者自行处理其中的复杂度
	
	4. Hint分片算法 `HintShardingAlgorithm` 用于处理使用Hint行分片的场景

### 自定义算法
这里我们使用`Complex`复合分片算法，可以通过多个字段进行分片处理

``` java
package com.zkxy.data.config;

import cn.hutool.core.date.DateField;
import cn.hutool.core.date.DateTime;
import cn.hutool.core.date.DateUtil;
import com.google.common.collect.Range;
import io.shardingsphere.api.algorithm.sharding.ListShardingValue;
import io.shardingsphere.api.algorithm.sharding.RangeShardingValue;
import io.shardingsphere.api.algorithm.sharding.ShardingValue;
import io.shardingsphere.api.algorithm.sharding.complex.ComplexKeysShardingAlgorithm;

import java.util.*;
import java.util.stream.Collectors;

public class FilterControlComplexShardingAlgorithm implements ComplexKeysShardingAlgorithm {
    @Override
    public Collection<String> doSharding(Collection<String> collection, Collection<ShardingValue> colValues) {
        for(ShardingValue shardingValue:colValues){
            String columnName=shardingValue.getColumnName();
            switch (columnName){
                case "yearMonth":
                    return shardingByMonth(shardingValue);
                case "departDate":
                    return shardingByDate(shardingValue);
            }
        }
        return collection;
    }

    /**
     * 根据时间范围进行分片
     * @return
     */
    private Collection<String> shardingByDate(ShardingValue shardingValue){
        List<String> tableNames = new ArrayList<>();
        /**
         * 范围 between
         */
        if(shardingValue instanceof RangeShardingValue){
            RangeShardingValue rangeShardingValue = (RangeShardingValue) shardingValue;
            Range valueRange = rangeShardingValue.getValueRange();
            Date begin = DateUtil.parse(valueRange.lowerEndpoint().toString());
            Date end = DateUtil.parse(valueRange.upperEndpoint().toString());
            for(DateTime dateTime:DateUtil.rangeToList(begin,end, DateField.MONTH)){
                tableNames.add(createTrueTableName(shardingValue,dateTime.month()+1));
            }
        }
        /**
         * 等于 in
         */
        if(shardingValue instanceof ListShardingValue){
            ListShardingValue listShardingValue = (ListShardingValue) shardingValue;
            for(Object value: listShardingValue.getValues()){
                tableNames.add(createTrueTableName(shardingValue,DateUtil.parse(value.toString()).month()+1));
            }
        }
        return tableNames;
    }




    /**
     * 根据月份进行分片
     * @param shardingValue
     * @return
     */
    private Collection<String> shardingByMonth(ShardingValue shardingValue){
        ListShardingValue listShardingValue= (ListShardingValue) shardingValue;
        return (Collection<String>) listShardingValue.getValues().stream()
                .map(o -> createTrueTableName(shardingValue,DateUtil.parse(o+"","yyyy-MM").month()+1))
                .collect(Collectors.toList());
    }

    /**
     * 生成真实物理表名
     * @param shardingValue
     * @param suffix
     * @return
     */
    private String createTrueTableName(ShardingValue shardingValue, Object suffix){
        return shardingValue.getLogicTableName()+"_"+suffix;
    }
}

```

##### 代码解析
- Collection<String> collection 执行`sql`中所有的逻辑表名
- Collection<ShardingValue> colValues 分片列信息

当执行`select`语句中`where`中包含配置的分片列且包含逻辑`beween and`、`in`、`=`3种逻辑时回调用我们自定义算法执行并返回需要查询的真实物理表名，如果没有分片按默认`actual-data-nodes`配置中的所有表执行相应`sql`

``` sql
# 分片列 yearMonth 对应会查询 filter_control_setting_10
select * from filter_control_setting where yearMonth = '2019-10-10';

# 无分片列 则会查询 filter_control_setting_1 .... filter_control_setting_12 一共12张表信息归并返回结果
select * from filter_control_setting;
```

### 遇到的问题
1. 最新版本 __4.0.0-RC1__ 使用 `ComplexKeysShardingAlgorithm`多字段复杂分片当遇到`between and`会报异常`BetweenRouteValue cannot be cast to ListRouteValue`，查看源码发现直接强转`ListRouteValue`导致报错，上`github`查看官方的`issues`也有同样的问题只有等待版本更新。
2. 数据库驱动版本过低也会导致集成失败，最好根据当前数据库版本更新较新的驱动`jar`包