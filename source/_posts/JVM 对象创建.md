---
title: JVM 对象创建过程
tags:
  - Jvm
categories:
  - Java
toc: false
date: 2019-04-17 21:01:28
---

![Java](/images/java.jpg)


>JVM 复习基本概念学习和记录

## 对象创建过程
![对象创建过程](/images/jvm-ObjectCreate.jpg)
- __类加载检查__：遇到`new`指令时，先会检查下常量池是否存在该类的符号引用，并检查是否`加载过`、`解析`、和`初始化`过，如果没有则进行类加载过程。

- __分配内存__：类加载检查通过，虚拟机会在`Java 堆`中为对象划分一块区域， 内存大小在类加载完成后确定，