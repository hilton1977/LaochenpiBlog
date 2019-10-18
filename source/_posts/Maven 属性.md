---
title: Maven 属性
tags:
  - Maven
categories:
  - 技术
toc: true
date: 2019-07-18 10:00:03
---

![](/images/maven.jpg)


##### Maven 内置属性
Maven 预定义,用户可以直接使用
- ${basedir} 表示项目根目录，即包含pom.xml文件目录
- ${version} 表示项目版本
- ${project.basedir} 同 ${basedir} 
- ${project.baseUri} 表示项目文件地址
- ${maven.build.timestamp} 表示项目构件开始时间
  
##### Maven POM 属性
使用 pom 属性可以引用到`pom.xml`文件对应元素的值
- ${project.build.directory} 表示主源码路径
- ${project.build.sourceEncoding} 表示主源码的编码格式
- ${project.build.sourceDirectory} 表示主源码路径
- ${project.build.finalName} 表示输出文件名称
- ${project.version} 表示项目版本,与 ${version} 相同

##### Maven 自定义属性
通过在`pom.xml`定义属性可以在其他地方引用通常在项目中用于定义引入`jar`的版本等
``` xml
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <version.spring>3.2.9.RELEASE</version.spring>
    <version.jackson>2.4.4</version.jackson>
    <java.version>1.7</java.version>
    <log4j2.version>2.6.2</log4j2.version>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
</properties>
```

##### 环境变量属性
所有的环境变量都可以用以`env.`开头的 Maven 属性引用，所有的环境变量都可以用以`env.`开头的Maven属性引用，${env.JAVA_HOME}表示JAVA_HOME环境变量的值

----

##### 参考如下：
http://maven.apache.org/guides/introduction/introduction-to-the-pom.html
http://maven.apache.org/pom.html
http://maven.apache.org/settings.html