---
title: Java 引用类型
tags:
  - 复习
categories:
  - Java
toc: false
date: 2020-08-26 15:40:50
---

### 强引用 StrongReference
最常见的引用类型 `Object o = new Object()`，特点当`JVM`内存不足时也不会被`GC`回收除非置为`null`，在一个内部方法有一个强引用时，引用保存在**方法栈**中，实际的内容会保存在**堆**中，当方法执行完毕后退出方法栈，引用对象的**引用数**变为**0**，这个对象则被`GC`回收 

**ArrayList 就是通过 clear 把底层数组循环置为 Null，让 gc 来进行回收工作**
``` java
/**
     * Removes all of the elements from this list.  The list will
     * be empty after this call returns.
     */
    public void clear() {
        modCount++;
        // clear to let GC do its work
        for (int i = 0; i < size; i++)
            elementData[i] = null;
        size = 0;
    }
```

### 软引用 SoftReference
如果一个对象具有软引用，只有当内存不足时才会进行回收，一般用于图片缓存、网页缓存等，例如在`spring-retry` 就有使用`SoftReferenceMapRetryContextCache`软引用作为重试上下文缓存，当内存不足的时候被`GC`回收，内存足够的时候避免创建浪费性能，也能避免大量创建引发**OOM**

``` java
Obejct a = new Object()
SoftReference soft = new SoftReference(a);
soft.get();
```

### 弱引用 WeckReference
弱引用描述被必须对象的，无论内存是否充足都会被`GC`所回收（未被强引用），`ThreadLocal`的实现就有用到软引用

``` java
@Data
@AllArgsConstructor
public class Test {
    private String name;

    public static void main(String[] args) throws InterruptedException {
        Test t = new Test("123");
        WeakReference weakReference = new WeakReference(t);
        System.gc();
        System.out.println(weakReference.get());
        t = null;
        System.gc();
        System.out.println(weakReference.get());
    }
}
```

>> 第一次输出 **Test(name=123)**，因为有强引用`t`指向，当把`t = null`执行完后进行垃圾回收则输出 **null**

### 虚引用 PhantomReference
无论是否被强引用、内存充足都会`GC`回收，在使用虚引用必须配合`ReferenceQueue`引用队列使用，它的主要作用是跟踪对象的垃圾回收的状态，当对象被回收后会将引用加入到引用队列中，以便后续进行其他行为操作，例如 `WeakHashMap`,其内部维护了`ReferenceQueue`当`Key`值被回收时，会删除相关的`Entry`
