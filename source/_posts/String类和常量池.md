---
title: String 类和常量池
tags:
  - 技术
categories:
  - Java
toc: false
date: 2019-03-28 16:09:24
---

![Java](/images/java.jpg)

> 面试经常会问到""创建的String和通过new String创建有什么不同。

## String 类和常量池
String 对象创建的2种方式
``` java
String str1 = "abcd";
String str2 = new String("abcd");
System.out.println(str1==str2);
```
使用`==`比较的是内存地址，虽然他们的值相同但是由于创建方式的原因它们的内存地址不一样，通过`""`方式创建的字符串存放在常量池中，通过`new`方式是在堆内存中创建一个新的对象，所有他们的内存地址不一样输出`false`。
![](/images/String==.jpg)

#### `注意：只要是通过new方式创建即会创建一个新的对象`

### String 类型的常量池比较
- 直接通过`""`声明出来的 String 对象会直接存储在常量池中。
- 如果不是通过`""`声明的 String 对象。可以使用`intern`方法，它是一个`Native`方法,在运行时常量池中如有值匹配的String字符串则返回，否则创建一个字符串并返回其引用。
``` java
String s1 = new String("计算机");
String s2 = s1.intern();
String s3 = "计算机";
System.out.println(s1 == s2);
System.out.println(s3 == s2);
```
输出 `false`、`true`，`s1` 属于堆中对象，`s2` 是常量池中对象,`s3` 属于常量池中对象



### String 字符串拼接
``` java
String str1 = "str";
String str2 = "ing";

String str3 = "str" + "ing";
String str4 = str1 + str2;  
String str5 = "string";
System.out.println(str3 == str4);
System.out.println(str3 == str5);
System.out.println(str4 == str5);
```
输出`false`、`true`、`false`，`str3`常量池对象、`str2`常量池对象、`str3`常量池对象、`str4`堆内对象、`str5`常量池对象
![](/images/String++.jpg)

## String s= new String("abc")生成了几个对象
面试经常会问到这样的问题，首先在常量池会生成`"abc"`对象，通过`new`会在堆中生成`abc`对象，所以答案是`2`个对象。

## 常量池拓展
`Java`基本类型的包装类的大部门实现了常量池技术，有`Byte`、`Short`、`Long`、`Charact`、`Boolean`。这5种包装类默认创建了`[-128,127]`的相应类型的缓存数据，超出范围的仍然需去创建新的对象。

以`Integer`为例
``` java
Integer i1 = 33;
Integer i2 = 33;
System.out.println(i1 == i2);// 输出true
Integer i11 = 333;
Integer i22 = 333;
System.out.println(i11 == i22);// 输出false
Double i3 = 1.2;
Double i4 = 1.2;
System.out.println(i3 == i4);// 输出false
```
代码中`Integer i1 = 33`在遍历时会把带代码封装成`Integer i1 = Integer.valueOf(33)`,从而使用的是常量池中的缓存对象。