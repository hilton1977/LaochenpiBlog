---
title: Redis 缓存问题场景
date: 2019-02-22 10:27:08
tags:
    - Redis
    - 技术
---

![](/images/redis.jpg)

>记录下学习Redis缓存实际项目中会出现的一些场景和解决方案，缓存穿透、缓存击穿、缓存雪崩

## Redis 缓存穿透
缓存穿透是指缓存和数据库都查询到不到，例如查询UserId=-1的用户，当大量类似访问请求发送到服务端，由于数据库一直无法查找到数据则缓存无法更新和插入，后续大量的请求全部落到了DB上。导致DB数据库压力增大发生崩溃、变慢。
##### 解决方案
- 在接口层面或者通过过滤器拦截器过滤掉一些恶意查询条件。
- 如果有查询不到的大量请求，可以设置`Key-Null`和`TTL`的时间设置30-60秒(具体根据实际业务需求来设定)，避免大量的后续恶意请求落在DB上。
![缓存穿透](/images/redis-caching-penetration.png)


## Redis 缓存雪崩
缓存雪崩是指缓存中大量的Key同一时间失效或缓存服务直接宕机导致大量的访问请求都落到了DB上，使得数据库压力过大导致连锁反应瘫痪宕机。
![缓存雪崩](/images/redis-caching-avalanche.png)

##### 失效解决：
- 热门数据缓存设置`TTL`延长或者永久
- 数据的缓存设置随机`TTL`防止同一时间失效

##### 服务宕机：
- Redis 高可用，使用主从+哨兵 `redis cluster`，避免全盘崩溃
- 本地 `ehcache` 缓存 + `hystrix` 限流/降级，避免DB被打死
- Redis 持久化，一旦重启立刻恢复数据
![解决方案](/images/redis-caching-avalanche-solution.png)

## 3.Redis 缓存击穿
缓存击穿是指同一个热门Key突然失效，大量的并发访问导致直接落在DB上，导致DB数据库压力增大宕机，与雪崩不同的是击穿是单一Key雪崩是大量热门Key。
- 数据的缓存`TTL`设置永久
- 使用互斥锁等待第一次请求缓存构建完成后释放锁，让其余所有请求直接通过缓存拿取数据。单机环境`Lock`类型，集群使用`Setnx`(set if not exits)



#### 查考资料 
https://github.com/doocs/advanced-java/blob/master/docs/high-concurrency/redis-caching-avalanche-and-caching-penetration.md
