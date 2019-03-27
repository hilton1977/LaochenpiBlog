---
title: SpringCloud 服务中心之 Eureka
date: 2018-08-01 16:43:15
categories: [Spring]
tags:
    - Spring
    - Java
---
![Eureka](/images/SpringCloud.jpg)

>SpringCloud微服务架构基于SpringBoot进行开发组件，即插即用非常方便，用了Spring Boot根本停不下来。SpringCloud包含了服务和注册中心(Zookeeper Eureka Consul)、熔断器(Hystrix)、动态路由(Zuul)、配置中心(Spring cloud config)、负责均衡(Ribbon)、REST服务调用(Fegin)等集成组件。让我们一步步通过项目来学习SpringCloud！

 ## 1. Eureka 服务发现和注册
Eureka 是 Netflix 旗下微服务开发组件，用于服务发现和注册中心，分为服务端和客户端，服务端作为注册中心作为其他客户端的提供注册服务，客户端将需要暴露的接口服务注册到服务端中，通过周期性向服务端发送心跳保证自身健康可用性。


## 2. EurekaServer 注册中心搭建
首先建立项目使用maven来构建项目，pom.xml依赖关系如下本项目用最新的版本进行教程，相关的官方教程可查看[Spring Cloud Eureka](http://projects.spring.io/spring-cloud/#quick-start)
##### pom.xml maven依赖配置
``` xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
 xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
<modelVersion>4.0.0</modelVersion>
<artifactId>EurekaServer</artifactId>

<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    <java.version>1.8</java.version>
</properties>

<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.1.RELEASE</version>
</parent>

<dependencyManagement>
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-dependencies</artifactId>
        <version>Finchley.SR1</version>
        <type>pom</type>
        <scope>import</scope>
    </dependency>
</dependencies>
</dependencyManagement>
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
</dependencies>
<!--项目构建maven插件-->
<build>
<plugins>
    <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
    </plugin>
</plugins>
</build>
</project>
```
##### SpringBoot 启动配置项
``` java
package com;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;

@SpringBootApplication
@EnableEurekaServer
@EnableWebSecurity
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class，args);
    }
}
```
##### WebSecurityConfig 安全认证配置
``` java
package com.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;

@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable(); //关闭csrf
        http.authorizeRequests().anyRequest().authenticated().and().httpBasic(); //开启认证
    }
}
```
##### application.yml 基本配置项
``` yml
#Eureka 服务中心设置 
eureka:
  client:
    #自身不注册
    register-with-eureka: false
    #是否开启检索服务
    fetch-registry: false
#security安全校验  
spring:
  security:
    user:
      name: root
      password: 123123
#服务器端口设置
server:
  port: 8888
```
启动项目通过 http://localhost:8888 查看Eureka注册中心管理页面，为了安全性加入了security安全校验，输入账号密码进入管理页面。
![Eureka Server Center](/images/eureka.png)

## 3.  EurekaClient 服务搭建
##### pom.xml maven依赖配置
``` xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <artifactId>EurekaClient</artifactId>
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
    </properties>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.1.RELEASE</version>
    </parent>
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Finchley.SR1</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```
##### SpringBoot 启动配置项
``` java
package com;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
@EnableEurekaClient
@RestController
public class Application {

    @RequestMapping("/test1")
    public String myTestService(){
        return "测试1";
    }

    @RequestMapping("/test2")
    public String myTestService2(){
        return "测试2";
    }

    public static void main(String[] args) {
        SpringApplication.run(Application.class，args);
    }
}
```
##### application.yml配置
``` yml
#  设置服务名
spring:
  application:
    name: EurekaClient1
#  设置注册中心地址 root:123123为注册中心设置的账号密码
eureka:
  client:
    service-url:
      defaultZone: http://root:123123@localhost:8888/eureka
```