---
title: Java8 Stream 流操作
date: 2019-03-29 10:06:00
categories: [Java]
comments: true
tags:
	- Java
	- 技术
---
![Java](/images/java.jpg)

>记录下 stream 流操作相关代码和一些细节问题。

## Stream(流)
`Stream` 流基本特性，不改变源数据、延迟执行、不存在数据，可简化代码可读性更高、美观、干净。有代码洁癖的小伙伴赶紧使用起来，它支持筛选、排序、聚合等，`Stream` 的聚合、消费、收集等操作只能进行一次。
![Stream 流](/images/java-stream.png)

### 常用方法
``` java
List<String> strings = Arrays.asList("3","1","2", "4");

# filter 筛选操作
int count = strings.stream()
		.filter(string -> string.isEmpty())
		.count();

# limit 数量限制
strings.stream()
	   .limit(5)
	   .forEach(System.out::println);

# map 将每个元素映射为其他
strings.stream()
       .map(string->string+"s")
       .collect(Collectors.toList());

# skip 忽略前N个元素
strings.stream()
	   .skip(2)
	   .collect(Collectors.toList())

# distinct 去重 非基本类型需要重写 hashcode equals 方法
strings.stream()
	   .distinct()
	   .forEach(System.out::println);

# sorted 排序 非基本类型需要重写 hashcode equals 方法
strings.stream()
       .sorted(Comparator.comparing(s -> s))
	   .forEach(System.out::println);
```