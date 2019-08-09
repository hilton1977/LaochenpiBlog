---
title: Linux 中判断条件
tags:
  - Linux
categories:
  - Linux
toc: false
date: 2019-08-08 10:38:34
---

![](images/linux-1.jpg)

> 学习记录下 Linux下判断条件 便于回来翻查

### if 基本语法
``` bash
if [comand];then 
   执行语句
elif[comand];then
   执行语句
else
   执行语句
fi
```

### 文件判断
[-d File] 存在且为文件夹
[-f File] 存在且为普通文件
[-s File] 存在且大小不为0
[-x File] 存在且为可执行
[-r File] 存在且为可读
[-w File] 存在且为写

### 字符串判断
[-z String] 是否为空长度为0
[-n String] 是否为非空 non-zero
[String] 是否为非空与[-n String]一样的功能
[String1 == String2] String 是否相等
[String1 != String2] String 是否不相等

### 数字判断
[Int1 -eq Int2] 是否相等
[Int1 -ne Int2] 是否不相等
[Int1 -gt Int2] 是否大于
[Int1 -lt Int2] 是否小于
[Int1 -ge Int2] 是否大于等于
[Int1 -le Int2] 是否小于等于
	
### 逻辑连接符
-a 且 &&
-o 或者 ||
! 非