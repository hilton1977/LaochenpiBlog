---
title: Redis的淘汰机制LRU算法
tags: []
categories: []
toc: false
date: 2020-09-14 14:46:09
---

![](/images/java.jpg)

### LRU 算法
LRU 全程 Least Recently Used（最近最久未使用），设计原则为：如果有一个数在一段时间未被访问，那么将来被访问的可能性就很小，当资源超过限制范围时应淘汰最久未被访问的数据

### LinkerHashMap 实现 LRU
`LinkerHsahMap`双向链表数据结构，在创建时可以选择`accessOrder`参数，默认`false`是按插入顺序进行排序，当我们设置`true`则按访问的顺序来进行排序

`LinkerHsahMap`调用`get()`方法会根据`accessOrder`调用`afterNodeAccess`方法将被访者置换尾部
``` java
void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            if (b == null)
                head = a;
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;
            ++modCount;
        }
  }
```
当插入新的数据会调用`removeEldestEntry`返回的boolean是否删除链表头部元素，重写该方法当缓存的长度大于限制则删除链表头部元素即删除了最久未被访问的元素，因为被访问过的元素会置换到链表的尾部
``` java
void afterNodeInsertion(boolean evict) { // possibly remove eldest
       LinkedHashMap.Entry<K,V> first;
        if (evict && (first = head) != null && removeEldestEntry(first)) {
            K key = first.key;
            removeNode(hash(key), key, null, false, true);
        }
}

protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return false;
}
```

我们经常用到的`Mybatis`内部也实现了LRUMap，核心就是重写`removeEldestEntry`方法删除链表头部元素

### 代码实现
``` java
public class LRUCache<K,V> extends LinkedHashMap<K,V> {
    private final Integer cacheSize;

    public LRUCache(Integer cacheSize) {
        super(cacheSize,0.75f,true);
        this.cacheSize = cacheSize;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size()>cacheSize;
    }

    public static void main(String[] args) {
        LRUCache c = new LRUCache(3);
        c.put("k1", 1);
        c.put("k2", 2);
        c.put("k3", 3);
   	c.get("k2");
        c.put("k4", 4);
     
        System.out.println(c);
    }
}
```

这里打印出来为 **{k3=3, k2=2, k4=4}**，当`get("k2")`时会将`k2`元素移到链表的末尾，下一步`put("k4")`时`removeEldestEntry`会返回`true`，缓存长度已超出限制范围删除首部元素所以打印出来就是 **{k3=3, k2=2, k4=4}**