---
title: 学习 CurrentHashMap
tags: []
categories: []
toc: false
date: 2020-05-25 20:36:32
---

![](/images/java.jpg)
> 平时工作多线程都用的少一面试问问CurrentHashMap 哦豁不会，场面一度陷入尴尬只知道多线程并发使用线程安全但是其中原理不知，现在学习 CurrentHashMap 的原理还有 HashMap 源码


### HashMap
HashMap 采用链地址法来存储数据，在查询插入数据效率特别高，根据`key`算出哈希地址得到哈希表对应数组下表，时间复杂度`O(1)`，JDK 1.7 采用数组+链表，JDK 1.8 采用 数组+链表+红黑树
![image.png](/images/2020/05/25/72d56dc0-9e9f-11ea-81d3-796c7949b71a.png)
#### hash 值算法
hashcode与(hashcode除以2的16次方)进行位异或运算得到新的hash值用于计算在数组中的下标位置
``` java
# 对 key 进行二进制转化位异或运算出 hash 值
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

#### put 方法
put 方法通过`hash()`方法得到对应`key`的值对应`Node`数组下标获取`Node<K,V>`，存在则进行`hash`比对链表或红黑树中的节点进行覆盖或者新增，其中当链表大于8时链表自动转化成红黑树
``` java
# 存放数据
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

# 存放数据方法
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    # 定义变量
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    # 判断是否未第一次存放数据，调用 resize 调整容量 返回新容量
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    # （n-1）& hash 数组长度与 hash 值进行位与运算得到数据下标中无 Node 数据则新增 Node
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    # 已有 Node 数据
    else {
        Node<K,V> e; K k;
        # 如果 Node 链表第一个Node 同 key 值覆盖
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        # 如果当前 Node 结构为红黑树加入树中
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        # Node 结构为链表循环尝试插入
        else {
            for (int binCount = 0; ; ++binCount) {
                # 链表下一个节点为空 覆盖插入的新节点
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    # 绑定次数大于等于7 即链表的长度即将超过8 需要讲链表转化成红黑树提交效率
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                # 如果 Node 链表中已有同 key 跳出循环
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                # Node 覆盖用于下次循环继续 next
                p = e;
            }
        }
        // 如果 Node 结构中已包含 key  进行值覆盖
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    # 修改次数自增
    ++modCount;
    # 当数组长度大于阈值进行扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

#### resize 扩容
Map的初始容量为16、负载因子为0.75f、最大容量限制为 2^30，当已有数据的长度大于阈值则进行扩容操作，每次扩容需要重新创建容量为之前2倍的Node数组进行转移，当数据量较大时反复进行扩容会消耗大量机器资源，所以应当在创建的时候根据实际情况给予合适的初始容量
``` java
# 调整大小
final Node<K,V>[] resize() {
    # 旧数据定义
    Node<K,V>[] oldTab = table;
    # 定义旧容量
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    # 旧阈值
    int oldThr = threshold;
    # 新容量和新阈值初始化 0 
    int newCap, newThr = 0;
    # 容量调整 如果旧容量大于0 
    if (oldCap > 0) {
        # 旧容量大于最大容量 （1<<30  2的30次方）
        if (oldCap >= MAXIMUM_CAPACITY) {
            # 更新阈值为 (1<<31)-1 2的31次方-1
            threshold = Integer.MAX_VALUE;
            # 返回旧数据
            return oldTab;
        }
        # 旧容量大于等于默认初始容量（16）且旧容量扩容两倍未超过最大容量时新阈值 x 2
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    # 旧阈值大于 0 旧阈值覆盖新容量
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    # 旧阈值小于等于0 新容量为默认初始化容量（16），新阈值为默认负责系列 0.75f * 16
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    # 新阈值为0时 重新计算阈值 当新容量小于最大容量且（新容量*负载因子）小于最大容量同时满足使用（新容量*负载因子）否则使用 Interger最大值
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    # 阈值赋值
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    # 根据新容量初始化新 Node
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];z
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

> 对于 hash(key)对应的数据下标采用了位与运算 hash(key)&(数组长度-1) ，等价于hash(key)%(数组长度)取模操作，得到的余数对应数组中的下标值

### CurrentHashMap 
`HashMap`是线程不安全，在多线程情况下操作会造成数据不一致等各类并发问题，对于线程安全有`HashTable`但是由于使用了`synchronized`同步锁住所有数据效率很低，`CurrentHashMap`相对于`HashTable`采用分段式锁更轻量效率更高

