---
title: Redis 数据类型及应用场景
tags:
  - Redis
categories:
  - Java
toc: false
date: 2019-04-10 16:19:30
---

![](/images/redis.jpg)

> 记录学习 Redis 的数据类型以及实际项目中的运用场景

## Redis 存储类型
`Redis` 面试经常会闻到支持那些存储类型，常用的类型有`String`、`Hash`、`List`、`Set`、`Sorted Set`等。

### String
`   Strings` 数据类结构是最简单的 `Key-value`类型，`Value` 的值根据执行的命令可为数值或字符串。
> 常用命令: `set,get,decr,incr,mget` 等。

__应用场景__：例如计数器功能`incr`可以运用于接口调用次数每次调用增加+1，配合`decr`每次减少-1，执行完毕会返回操作之后的值。


### Hash
`Hash` 键值对格式，适合存放对象信息例如用户信息，用户名ID对应的用户信息`value`。有时候我们使用的是序列化对象取出和存在都需要序列化消耗性能。
> 常用命令： `hget，hset，hgetall，Hash` 实现有2种在数据量较小时会采用类似一维数组紧凑存储,对应的`value`的`redisObject`的`encoding`为`zipmap`，当数据足够大时内部会自动转化成真正`HashMap`结构，`encoding`为`ht`。
  
__应用场景__：例如用户信息、后台列表信息、用户的权限信息等。

### List
`List` 链表，`Redis` 实现为一个双向链表，即可以反向查询和遍历，更方便操作但也带来更多的性能消耗。
> 常用命令: `lpush，rpush，lpop，rpop，lrange，blpop`等

__应用场景__：例如使用`lrange`可以做分页操作，一些列表信息展示，使用`ltrim`限制长度可限制最新N条数据。`lpsuh`、`rpush`添加数据，`lpop`、`rpop`删除数据。

### Set
`Set` 类似`List`的功能，主要功能可以去重当你不希望有重复数据可以使用，并且可以判断是否某数据是否在集合内还可以处理2个`Set`交集、并集、差集。
> 常用命令： `sadd，spop，smembers，sunion`等，实现方式是`value`永远为`null`的`HashMap`，通过计算`Hash`的方式来快速排重。

__应用场景__：例如用户权限公共的权限交集，微博里的共同好友、共同关注等。

### SortedSet
当你需要没有重复数据且有序则可以使用`SortedSet`，不同的是每个元素都会关联一个double类型的分数，redis正是通过分数来为集合中的成员进行从小到大的排序。
> 常用命令: `zadd，zcard，zrem`

### Geo
用于存放地理坐标


### 代码运用
``` java

@RunWith(SpringRunner.class)
@SpringBootTest
public class TestControllerTest {

    @Resource
    private StringRedisTemplate stringRedisTemplate;

    private final String stringKey = "stringKey";

    private final String listKey = "listKey";

    private final String hashKey = "hashKey";

    private final String geoType = "geoKey";

    private final String setType1 = "setType1";

    private final String setType2 = "setType2";

    private final String sortedType = "sortedType";


    @Test
    /**
     * 常见的string类型数据
     */
    public void stringType() {
        ValueOperations<String, String> valueOperations = stringRedisTemplate.opsForValue();
        valueOperations.set(stringKey, "test");
        System.out.println(valueOperations.get(stringKey));
        stringRedisTemplate.delete(stringKey);
    }

    @Test
    /**
     * list数据类型 可分页 可作为简单的队列
     */
    public void listType() {
        List<String> pushList = Arrays.asList("push3", "push2", "push1");
        ListOperations<String, String> listOperations = stringRedisTemplate.opsForList();
        listOperations.leftPushAll(listKey, pushList);
        System.out.println(listOperations.size(listKey));
        String str1 = listOperations.rightPop(listKey);
        String str2 = listOperations.rightPop(listKey, 10, TimeUnit.MILLISECONDS);
        String str3 = listOperations.rightPop(listKey);
        System.out.println(str1);
        System.out.println(str2);
        System.out.println(str3);
    }

    @Test
    /**
     * hash 键值对类型
     */
    public void hashType() {
        HashOperations<String, String, String> hashOperations = stringRedisTemplate.opsForHash();
        hashOperations.put(hashKey, "1", "1str");
        hashOperations.put(hashKey, "2", "2str");
        hashOperations.put(hashKey, "3", "3str");
        Map<String,String> objectMap = hashOperations.entries(hashKey);
        for (String o : objectMap.values()) {
            System.out.println(o);
        }
        System.out.println(hashOperations.keys(hashKey));
        System.out.println(hashOperations.get(hashKey, "1"));
    }

    @Test
    /**
     * Geo常用语地理位置 坐标 地点距离
     */
    public void geoType() {
        GeoOperations<String, String> geoOperations = stringRedisTemplate.opsForGeo();
        geoOperations.add(geoType, new Point(123.02, 232.0), "beijing");
        geoOperations.add(geoType, new Point(1232.0222, 423.23), "shanghai");
        Distance distance = geoOperations.distance(geoType, "beijing", "shanghai");
        System.out.println(distance.getValue());
    }

    @Test
    /**
     * 不可重复set
     */
    public void setType() {
        SetOperations<String, String> setOperations = stringRedisTemplate.opsForSet();
        setOperations.add(setType1, "1","3", "4");
        setOperations.add(setType2, "2","4","5");
        //差集
        Set<String> differenceStr = setOperations.difference(setType2,setType1);
        System.out.println(differenceStr);
        //交集
        Set<String> intersectStr = setOperations.intersect(setType1, setType2);
        System.out.println(intersectStr);
    }

    @Test
    /**
     * 不可重复且通过分数进行排序set
     */
    public void sortedSetType() {
        ZSetOperations<String, String> zSetOperations = stringRedisTemplate.opsForZSet();
        zSetOperations.add(sortedType, "小于", 72.2);
        zSetOperations.add(sortedType, "小明", 52.2);
        zSetOperations.add(sortedType, "小方", 22.5);
        zSetOperations.add(sortedType, "小球", 99.2);
        // 升序分数排名 从0 开始
        System.out.println(zSetOperations.rank(sortedType,"小于"));
        // 降序分数排名 从0 开始
        System.out.println(zSetOperations.reverseRank(sortedType,"小于"));
        //50-80 分数 升序 values
        Set<String> scores=zSetOperations.rangeByScore(sortedType,50,80);
        System.out.println(scores);
        //20-80 分数 降序 values
        Set<String> reverseScores=zSetOperations.reverseRangeByScore(sortedType,20,80);
        System.out.println(reverseScores);

    }

}
```

## 过期策略
- 定时删除 对于带有`TTL`标示的`ke`， Redis 会定时随机抽取检查是否过期并删除
- 惰性删除 当你去获取一个`key`时如果超时则会直接删除该key且没有返回
对于以上2种删除并不能覆盖到所有的`key`，当这些数据后续不再使用会浪费大量内存空间属于无效缓存，如果内存不足时想继续存入新数据则会产生与预期不符的结果，`Redis`对于内存不足情况下提供了内存淘汰机制

#### 内存淘汰机制
- __noeviction__ 内存不足写入操作直接报错返回错信息
- __allkey-lru__ 所有key通用，优先删除最近最少使用(less recently used ,LRU) 的 key
- __allkey-random__ 所有key通用，随机删除一部分 key
- __volatile-lru__ 只限于设置了 expire 的部分，随机删除一部分 key
- __volatile-random__ 只限于设置了 expire 的部分，优先删除剩余时间(time to live TTL) 短的key