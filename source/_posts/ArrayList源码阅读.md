---
title: ArrayList源码阅读
date: 2019-03-18 21:01:48
categories: [Java]
tags:
    - Java
---

> 记录学习回顾Java基础学习源码思想ArrayList，平时光顾着写业务代码基础细节都没有进行积累导致出去面试被人家一顿虐，只注重外功不注重内功是不行的。

### ArrayList
平时最常用的集合，特点有序查找效率高`线程不安全`底层是数组实现了动态数组的功能，实现了`RandomAccess`(快速随机访问)、`Cloneable`(克隆接口)、`Serializabele`(序列化)等接口。

#### 源码解析
``` java

public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
    private static final long serialVersionUID = 8683452581122892189L;

    /**
     * 默认的初始化容量 10
     */
    private static final int DEFAULT_CAPACITY = 10;

    /**
     * 共享的静态空Object数组用于空实例
     */
    private static final Object[] EMPTY_ELEMENTDATA = {};

    /**
     * 共享的静态空数组实例 用于最常用的new ArrayList() 无参实例使用
     */
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     * 用于存放加入的数据数组 transient 关键字用于标记不需要序列化的字段
     */
    transient Object[] elementData; // non-private to simplify nested class access

    /**
     * 
     * 整个数组的长度 size 即size()返回值
     * @serial
     */
    private int size;

    /**
     * 有参数的构造函数 initialCapacity 用于给集合初始化容量
     */
    public ArrayList(int  initialCapacity) {
    	//初始化一个大小为 initialCapacity 的Object数组
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
        	//如果初始容量为0使用静态 EMPTY_ELEMENTDATA 默认的空数组
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }

    /**
     * 最常用的初始化方法
     */
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

    /**
     * Collection 传入一个集合元素列表 E为泛型 指定传入的集合类型
     */
    public ArrayList(Collection<? extends E> c) {
    	//集合转化为数组 并初始化elementData
        elementData = c.toArray();
        //初始化size的值
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            // 由于传入的集合真实类型不一样所以需要调用 Arrays.copyOf 复制到一个新的Object[]数组中，以便可以存放任意类型
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }

    /**
     *修改当前容器值为实际元素的个数
     */
    public void trimToSize() {
        modCount++;
        if (size < elementData.length) {
            elementData = (size == 0)
              ? EMPTY_ELEMENTDATA
              : Arrays.copyOf(elementData, size);
        }
    }

    /**
     * 自行控制扩容大小 
     * 如果扩容值大于默认值10 则按传入值进行扩容处理判断
     */
    public void ensureCapacity(int minCapacity) {
        int minExpand = (elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
            // any size if not default element table
            ? 0
            // larger than default for default empty table. It's already
            // supposed to be at default size.
            : DEFAULT_CAPACITY;

        if (minCapacity > minExpand) {
            ensureExplicitCapacity(minCapacity);
        }
    }

    /**
     * 计算最小容量
     */
    private static int calculateCapacity(Object[] elementData, int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        return minCapacity;
    }

    /**
     * 根据minCapacity进行扩容
     */
    private void ensureCapacityInternal(int minCapacity) {
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
    }

    /**
     * 判断是否需要进行扩容操作 如果扩容值大于实际的数组长度则进行扩容
     */
    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }

    /**
     * 能分配的最大的数组大小 Integer数值最大值(2^31-1)-8
     */
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    /**
     * 扩容的核心代码
     * 每次扩容的大小为 当前数组长度+(数组长度/2)
     * 如果扩容新容量小于需要扩容量值则覆盖新容量值
     * 如果扩容新容量大于MAX_ARRAY_SIZE则直接使用Interger.MAX_VALUE否则使用MAX_ARRAY_SIZE
     */
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

    /**
     *当需要扩容大于MAX_ARRAY_SIEZ或小于0 返回合适值
     */
    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
}

```