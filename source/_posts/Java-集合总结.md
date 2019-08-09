---
title: Java 集合总结
tags:
  - 复习
categories:
  - Java
toc: false
date: 2019-07-22 16:43:25
---

![Java](/images/java.jpg)

> 记录回顾下 Java 集合内容，面试会经常会问到各种集合的特点和区别，在不同情况下合理使用集合处理数据


## Java 集合
Java 集合分为两大接口`Collection`（元素集合）、`Map`（键值对集合），根据两大接口又分为各类集合

![](/images/collection.png)

### Map 键值对
Map 由 key 和 value 键值对组成，key 不重复存入重复的 key 会覆盖原有 value

|集合|数据结构|是否有序|线程安全|特点|
|:-|:-:|:-:|:-:|:-:|
|HashMap|数组和链表|无序|不安全|链表长度大于阈值8则转化成红黑树，小于6则转回链表|
|TreeMap|red-black（红-黑）树|无序|不安全|映射根据其键的自然顺序进行排序，或者根据创建映射时提供的 Comparator 进行排序|
|LinkedHashMap|链表|有序|不安全|允许使用 Null 键值|

### Collection 集合

#### List 接口
List 代表有序的 Collection 且元素可以重复，用户对插入的元素位置精确把控，数组维护整个集合顺序，同时可以通过 index 下标直接访问相应的元素

|集合|数据结构|是否有序|线程安全|特点|
|:-|:-:|:-:|:-:|:-:|
|ArrayList|数组|有序|不安全|随机访问、查询效率高|
|Vector|数组|有序|安全||
|LinkedList|双向链表|有序|不安全|插入删除效率高，可从头尾顺序进行插入删除|

#### Set 接口
Set 代表无序的 Collection 且不包含重复元素，可以存入有且只有一个`Null`元素

|集合|数据结构|是否有序|线程安全|特点|
|:-|:-:|:-:|:-:|:-:|
|HashSet|简化版 HashMap|无序|不安全|内部由 HashMap 实现 key 为存入数据 Value 都为同一个 Object 对象，HashMap 的 key不可重复特性实现去重效果|
|TreeSet|TreeMap 实现|无序|不安全|内部由 TreeMap 实现的有序二叉树，基本类型按照自然顺序排序，对象则根据 Comparator 进行自动排序|
|LinkedHashSet|双向链表|有序|不安全|HashSet 子类内部由链表来维护插入元素顺序|