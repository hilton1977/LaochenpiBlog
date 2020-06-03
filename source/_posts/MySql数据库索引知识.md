---
title: MySql数据库索引知识
tags:
  - Java
originContent: ''
categories:
  - 数据库
toc: false
date: 2020-05-27 21:02:41
---

![](/images/MySQL.jpg)

### 索引
- 唯一索引： 索引列的值必须唯一，但允许有空值。如果是组合索引，则列值的组合必须唯一。
- 主键索引： 特殊的唯一索引，一张表有且仅有一个主键索引，不允许为空值，提高搜索效率并提供唯一约束
- 组合索引： 多个字段上创建的索引，注意只有在查询条件中包含了第一个字段才会使用索引，使用组合索引遵循左前缀集合

https://www.cnblogs.com/developer_chan/p/9208404.html