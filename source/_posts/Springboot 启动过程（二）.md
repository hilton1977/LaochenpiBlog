---
title: SpringBoot 启动源码解析（二）之监听器
tags:
  - Spring
categories:
  - Spring
toc: false
date: 2020-05-11 14:14:36
---

![](/images/spring.jpg)
> 监听器在 Spring 中特别重要可以自定义监听器获取各类事件做一些特殊操作

#### SpringApplicationRunListeners 事件监听器
将存于`MATE/spring.factories`中的`SpringApplicationRunListener`类型全部实例化，传入`SpringApplication`对象作为参数
``` java
 private SpringApplicationRunListeners getRunListeners(String[] args) {
        Class<?>[] types = new Class[]{SpringApplication.class, String[].class};
        return new SpringApplicationRunListeners(logger, this.getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args));
}
```
以`EventPublishingRunListener`（事件推送监听器）为例，将之前已实例化`ApplicationListener`加入`SimpleApplicationEventMulticaster`（事件多广播器）的`defaultRetriever`(检索器) 中待事件发生遍历推送

``` java
public EventPublishingRunListener(SpringApplication application, String[] args) {
    this.application = application;
    this.args = args;
    // 初始化事件广播器
    this.initialMulticaster = new SimpleApplicationEventMulticaster();
    // 遍历监听器集合并加入事件广播器中
    Iterator var3 = application.getListeners().iterator();
    while(var3.hasNext()) {
        ApplicationListener<?> listener = (ApplicationListener)var3.next();
        this.initialMulticaster.addApplicationListener(listener);
    }
}

public void addApplicationListener(ApplicationListener<?> listener) {
    synchronized(this.retrievalMutex) {
        Object singletonTarget = AopProxyUtils.getSingletonTarget(listener);
        if (singletonTarget instanceof ApplicationListener) {
            this.defaultRetriever.applicationListeners.remove(singletonTarget);
        }
        this.defaultRetriever.applicationListeners.add(listener);
        // 清空检索器缓存
        this.retrieverCache.clear();
    }
}
```

#### ApplicationEvent 事件
在 Spring 的启动的个个阶段会推送不同的 ApplicationEvent 事件至监听器完成相应阶段的逻辑操作
- **ApplicationStartingEvent**：在 Environment 和 ApplicationContext 可用之前 & 在 ApplicationListener 注册之后发布。
- **ApplicationEnvironmentPreparedEvent** ：配置环境完毕时推送，获取到当前配`ConfigurableEnvironment`可自定义修改配置或新增额外配置
- **ApplicationContextInitializedEvent**：在 bean 定义加载之前 & ApplicationContextInitializers 被调用之后 & ApplicationContext 开始准备之后发布
- **ApplicationPreparedEvent**：在 ApplicationContext 完全准备好并且没有刷新之前发布，此时 bean 定义即将加载， 
 Environment 已经准备好被使用。
- **ApplicationStartedEvent**：在 ApplicationContext 刷新之后，调用 ApplicationRunner 和 CommandLineRunner 之前发布
- **ApplicationReadyEvent**：应用已经准备好接受请求时发布。

#### SimpleApplicationEventMulticaster 事件广播器
![simpleApplicationEventMulticaster](/images/simpleApplicationEventMulticaster.png)
每次事件发生内部调用`multicastEvent`方循环调用监听器`onApplicationEvent`，如果有线程池则异步调用否则同步调用，事件监听推送就是经典的__观察者设计模式__
``` java
public void multicastEvent(ApplicationEvent event, @Nullable ResolvableType eventType) {
    // 可解决类型
    ResolvableType type = eventType != null ? eventType : this.resolveDefaultEventType(event);
    // 根据事件类型和可解决类型过滤已有监听器
    Iterator var4 = this.getApplicationListeners(event, type).iterator();

    while(var4.hasNext()) {
        ApplicationListener<?> listener = (ApplicationListener)var4.next();
        // 获取线程池
        Executor executor = this.getTaskExecutor();
        if (executor != null) {
            executor.execute(() -> {
                this.invokeListener(listener, event);
            });
        } else {
            this.invokeListener(listener, event);
        }
    }
}
```

当事件发生并不是所有的监听器都需要推送，通过父类`AbstractApplicationEventMulticaster`中`getApplicationListeners`方法过滤集合并以`AbstractApplicationEventMulticaster.ListenerCacheKey`作为 key `AbstractApplicationEventMulticaster.ListenerRetriever` 作为 value 的`ConcurrentHashMap`检索器缓存用于各种事件检索，`supportsEvent`代码中先判断当前监听器类型如果为`GenericApplicationListener`则调用`supportsEventType`和`supportsSourceType`方法判断是否都为`true`否则通过`GenericApplicationListenerAdapter`适配器进行包装转化后判断

``` java
protected Collection<ApplicationListener<?>> getApplicationListeners(ApplicationEvent event, ResolvableType eventType) {
    Object source = event.getSource();
    Class<?> sourceType = source != null ? source.getClass() : null;
    // 实例化监听器缓存 key
    AbstractApplicationEventMulticaster.ListenerCacheKey cacheKey = new AbstractApplicationEventMulticaster.ListenerCacheKey(eventType, sourceType);
    // 根据 key 获取相应监听器检索器
    AbstractApplicationEventMulticaster.ListenerRetriever retriever = (AbstractApplicationEventMulticaster.ListenerRetriever)this.retrieverCache.get(cacheKey);
    // 存在直接返回否则进行判断过滤生成缓存
    if (retriever != null) {
        return retriever.getApplicationListeners();
    } else if (this.beanClassLoader == null || ClassUtils.isCacheSafe(event.getClass(), this.beanClassLoader) && (sourceType == null || ClassUtils.isCacheSafe(sourceType, this.beanClassLoader))) {
        synchronized(this.retrievalMutex) {
            // 获取监听器检索器缓存
            retriever = (AbstractApplicationEventMulticaster.ListenerRetriever)this.retrieverCache.get(cacheKey);
            if (retriever != null) {
                return retriever.getApplicationListeners();
            } else {
                // 实例化一个标识已过滤的监听器检索器
                retriever = new AbstractApplicationEventMulticaster.ListenerRetriever(true);
                // 根据事件类型和来源类型进行检索过滤
                Collection<ApplicationListener<?>> listeners = this.retrieveApplicationListeners(eventType, sourceType, retriever);
                // 将该事件类型的检索器加入 Map 缓存
                this.retrieverCache.put(cacheKey, retriever);
                return listeners;
            }
        }
    } else {
        return this.retrieveApplicationListeners(eventType, sourceType, (AbstractApplicationEventMulticaster.ListenerRetriever)null);
    }
}

// 判读是否支持该事件类型以及来源
protected boolean supportsEvent(ApplicationListener<?> listener, ResolvableType eventType, @Nullable Class<?> sourceType) {
    GenericApplicationListener smartListener = listener instanceof GenericApplicationListener ? (GenericApplicationListener)listener : new GenericApplicationListenerAdapter(listener);
    return ((GenericApplicationListener)smartListener).supportsEventType(eventType) && ((GenericApplicationListener)smartListener).supportsSourceType(sourceType);
}
```

>>  在 SimpleApplicationEventMulticaster 实例化时会将默认的 defaultRetriever 引用赋值给 `retrievalMutex` 检索互斥体，在内部很多方法都使用了 synchronized(this.retrievalMutex) 同步锁来保证线程安全