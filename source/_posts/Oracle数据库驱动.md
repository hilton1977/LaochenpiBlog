---
title: Oracle数据库驱动
tags:
  - Java
categories:
  - 数据库
toc: false
date: 2019-10-17 14:40:59
---

> 记录下 Ｏracle 版本和 jdbc 一些对应知识

#### 查询数据库版本
``` sql
# 版本信息
select* from v$version;
# 产品信息
select * from product_component_version;
``` 

#### 项目引入
一般项目使用maven 来管理项目jar 在引入 ojdbc 时会无法加载下载必须要去官方自行下载然后注册到 `C:\Users\Administrator\.m2\repository\com\oracle\数据库版本 ` (ps:此地址为默认的maven库地址)，下
载地址为 https://www.oracle.com/database/technologies/jdbcdriver-ucp-downloads.html 自行根据情况下载相应的 ojdbc.jar

``` bash
mvn install:install-file -Dfile=D:\Downloads\ojdbc6.jar -DgroupId=com.oracle -DartifactId=ojdbc6 -Dversion=11.2.0.3 -Dpackaging=jar -DgeneratePom=true
```

##### 参数详解 
- install:可以将项目本身编译并打包到本地仓库
- install-file:安装文件
- -Dfile=D:\Downloads\ojdbc6.jar : 指定要打的包的文件位置
- -DgroupId=com.oracle : 指定当前包的groupId为com.oracle
- -DartifactId=ojdbc6 : 指定当前的artifactfactId为ojdbc6
- -Dversion=11.2.0.3 : 指定当前包的版本为11.2.0.3
- -DgeneratePom=true:是否生成pom文件

```
# maven 引入
<dependency>
   <groupId>com.oracle</groupId>
   <artifactId>ojdbc6</artifactId>
   <version>11.2.0.3</version>
</dependency>
```

#### 版本对应的 jdbc
![](/images/oralce_jdbc.png)