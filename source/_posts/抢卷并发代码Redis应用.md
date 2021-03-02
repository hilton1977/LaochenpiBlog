---
title: 抢卷并发代码Redis应用
tags:
  - 技术
categories:
  - 代码块
toc: false
date: 2020-09-28 11:12:10
---

### 代码实现
最近的项目有一个抢卷业务为保证性能使用`redis`来实现保证高并发性能，首先分析下抢卷业务
- 优惠卷的数量不能超发，保证卷的一人一张
- 锁的粒度尽可能的小
- 锁的维度唯一性（用户id+卷id）
- 优惠卷的数量事先写入`Redis`防止大量访问`DB`获取卷数量，`DB`由于数据的隔离性读取的数量也可能会不同发生超发场景

#### Redisson
通过注入`RedissonClient`来实现各种锁场景
``` xml
<dependency>
     <groupId>org.redisson</groupId>
     <artifactId>redisson</artifactId>
     <version>3.9.1</version>
</dependency>
```

### 代码流程
1. 尝试获取锁（用户id+卷id）唯一锁
2. 获取成功开始进行抢卷操作否则抢卷失败
3. 进行`DB`操作插入用户卷关联关系，插入成功进行缓存写入修改卷的数量和用户已抢卷的key防止用户多次抢卷

``` java
    @Override
    @Transactional(rollbackFor = Exception.class)
    public boolean getCoupon(String userId, String materielId) throws Exception {
        // 以优惠卷为维度加锁 防止超发
        String ticketLock = new StringBuilder(RedisPrefixEnum.LIVE_TICKET.getPrefix())
                .append(materielId).toString();
        // 用户领卷 key
        String tickerUser = new StringBuilder(RedisPrefixEnum.LIVE_COUPON_USER_LOCK.getPrefix())
                .append(materielId).append("_").append(userId).toString();

        SysUserEntity user = sysUserService.getById(userId);
        boolean result = redisLocker.fairLock(ticketLock, () -> {
            Integer ticketSum = checkTicket(userId, materielId);
            // 领卷逻辑
            PortalMateriel portalMateriel = portalMaterielMapper.selectById(materielId);
            PortalUserMateriel userMateriel = new PortalUserMateriel();
            userMateriel.setId(UUIDUtils.get32UUID());
            userMateriel.setUserId(userId); // 助力活动发起人
            userMateriel.setMaterielId(materielId);
            userMateriel.setTel(user.getTel());
            userMateriel.setMaterielCode(portalMateriel.getPre() + DateUtil.current(true));
            userMateriel.setTitle(portalMateriel.getTitle());
            userMateriel.setParValue(portalMateriel.getParValue());
            userMateriel.setEndDate(portalMateriel.getEndDate());
            userMateriel.setStatus(0); // 未使用
            userMateriel.setType(portalMateriel.getType()); // 优惠券
            // 插入用户物件表
            portalUserMaterielMapper.insert(userMateriel);
            // 更新优惠卷数量缓存
            redisService.setModel(ticketLock, ticketSum - 1);
            // 新增领卷记录 key 防止用户多次领取
            redisService.setModel(tickerUser, null);
            return true;
        }, 60);
        return result;

    }
```
这里采用了`Redisson`组件方便使用各种锁，用户抢卷为了保证公平所有使用 **fairLock公平锁** ，当大量用户进行抢卷操作时根据等待时长顺序依次进行抢卷逻辑，首先卷的数量需要先缓存到`redis`中防止在高并发下直接访问`DB`从而影响正常使用

在查阅了资料和同事的指导下对上述代码进行重构优化，采用`Redis`的原子性操作`decr`自减方法，锁的粒度也是以人为单位，卷的数量通过原子减是否小于等于0来判断用户领卷是否成功，使用`lock`锁对用户进行加锁，防止用户多次点击抢卷或通过其他方式请求抢卷接口，特别要注意设置锁的**释放时间**防止死锁导致后续所有请求阻塞产生系统严重性能问题，有异常后finally中必写`lock.unlock()`释放锁操作


``` java
    @Override
    @Transactional(rollbackFor = Exception.class)
    public boolean getCoupon(String userId, String materielId) throws Exception {
        if (this.hasCoupon(userId, materielId)) {
            throw new GlobalException("你已经领过券了");
        }
        String userKey = userKey(userId, materielId).toString();
        // 尝试获取锁 防止当前用户疯狂点击
        RLock lock = redissonClient.getLock(userKey);
        lock.tryLock(0L,20L, TimeUnit.SECONDS);
        if (!lock.isLocked()) {
            throw new GlobalException("请勿重复领取");
        }
        try {
            // 先尝试领卷 原子性减少卷数量 -1
            Long sum = getCouponNum(materielId);
            if (sum <= 0) {
                throw new GlobalException("很遗憾，券已领完！");
            }
            // 发生异常需要退还卷
            PortalUserMateriel portalUserMateriel = initPortalUserMaterielByUserId(userId, materielId);
            int saveNum  = portalUserMaterielMapper.insert(portalUserMateriel);
            // 保存失败抛出异常执行退还操作
            if (saveNum != 1) {
                throw new GlobalException("抢卷失败");
            }

        } catch (Exception e) {
            log.error(userId+"用户抢卷"+materielId+"失败",e);
            throw e;
        } finally {
            // 最终释放锁
            lock.unlock();
        }
        // 领取成功插入领取记录
        redisService.setModel(userKey, null);
        return true;

    }
```

如果对时效性不高可以使用队列去做抢卷的逻辑，这样可以保证大量的请求进行分发处理防止瞬间的大量请求拖垮服务器