---
title: fast-fail 和 fast-safe的区别
tags:
  - Java
categories:
  - Java
toc: false
date: 2019-07-24 11:25:10
---

![Java](/images/java.jpg)

> 回头复习下一些面试常见的一些问题 fast-fail 和 fast-safe 两种机制

### fast-fail
fast-fail 快速失败，在`java.util`包下的集合进行迭代遍历时会调用`checkForComodification`方法检查，当`modCount != expectedModCount`时会抛出`ConcurrentModificationException`异常
``` java
 final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
 }
```

#### 注意
在多线程环境下并发操作集合可能会导致`modCount`与`expectedModCount`的值和预期不一致导致条件判断通过不会抛出异常，因此在多线程环境下应该使用`java.concurrent`包下的集合


### fast-safe
fast-safe 安全失败，任何对集合结构的修改都会在一个复制的集合上进行修改，因此不会抛出`ConcurrentModificationException`

#### 注意
由于 fast-safe 遍历是采用了复制原有集合内容作为一个新的集合进行遍历，所以会带来更大的内存消耗且原始数据发现变化后是不能被迭代器所检测发现数据变化


||Fail Fast Iterator|Fail Safe Iterator|
|-|-|-|
|Throw ConcurrentModificationException|Yes|No|
|Clone Object|No|Yes|
|Memory OverHead|No|Yes|
|Examples|ArrayList、HashMap、HashSet|CopyOnWriteArrayList、ConcurrentHashMap|