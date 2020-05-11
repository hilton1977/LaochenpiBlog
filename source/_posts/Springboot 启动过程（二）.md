---
title: SpringBoot 启动源码解析（二）
tags:
  - Spring
categories:
  - Spring
toc: false
date: 2020-05-11 14:14:36
---

![](/images/spring.jpg)
> SpringBoot 启动源码解析（二） 看看 SpringApplication run 方法干了啥

## SpringBoot run
SpringApplication `run` 方法就是正式开始环境配置刷新容器等操作

``` java
public ConfigurableApplicationContext run(String... args) {
    // 监控器实例化
    StopWatch stopWatch = new StopWatch();
    // 启动监控器
    stopWatch.start();
    ConfigurableApplicationContext context = null;
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList();
    // 无头配置初始化
    this.configureHeadlessProperty();
    // 初始化 SpringApplicationRunListeners 并开启
    SpringApplicationRunListeners listeners = this.getRunListeners(args);
    listeners.starting();
    Collection exceptionReporters;
    try {
        // 初始化应用参数
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);	
        // 初始化配置环境 （property）
        ConfigurableEnvironment environment = this.prepareEnvironment(listeners, applicationArguments);
        this.configureIgnoreBeanInfo(environment);
        Banner printedBanner = this.printBanner(environment);
        context = this.createApplicationContext();
        // 初始化异常上报
        exceptionReporters = this.getSpringFactoriesInstances(SpringBootExceptionReporter.class, new Class[]{ConfigurableApplicationContext.class}, context);
        // 准备 刷新 容器
        this.prepareContext(context, environment, listeners, applicationArguments, printedBanner);
        this.refreshContext(context);
        this.afterRefresh(context, applicationArguments);
        stopWatch.stop();
        if (this.logStartupInfo) {
            (new StartupInfoLogger(this.mainApplicationClass)).logStarted(this.getApplicationLog(), stopWatch);
        }
        
        listeners.started(context);
        this.callRunners(context, applicationArguments);
    } catch (Throwable var10) {
        this.handleRunFailure(context, var10, exceptionReporters, listeners);
        throw new IllegalStateException(var10);
    }

    try {
        listeners.running(context);
        return context;
    } catch (Throwable var9) {
        this.handleRunFailure(context, var9, exceptionReporters, (SpringApplicationRunListeners)null);
        throw new IllegalStateException(var9);
    }
}
```

#### SpringApplicationRunListeners 事件监听器
SpringApplicationRunListeners 实例化就是`EventPublishingRunListener`将所有已有监听器加入`SimpleApplicationEventMulticaster`的`defaultRetriever`(默认检索器中)
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
ApplicationEvent 事件分为以下几种


#### SimpleApplicationEventMulticaster 事件广播器
![simpleApplicationEventMulticaster](/images/simpleApplicationEventMulticaster.png)
`SimpleApplicationEventMulticaster` 每次事件发生内部调用`multicastEvent`方循环调用监听器`onApplicationEvent`，如果有线程池则异步调用否则同步调用，事件监听推送就是经典的__观察者模式__
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

通过父类`AbstractApplicationEventMulticaster`中`getApplicationListeners`方法过滤需推送`ApplicationListener`集合并以`AbstractApplicationEventMulticaster.ListenerCacheKey`作为 key `AbstractApplicationEventMulticaster.ListenerRetriever` 作为 value 的`ConcurrentHashMap`检索器缓存用于各种事件检索，`supportsEvent`代码中先判断当前监听器类型如果为`GenericApplicationListener`则调用`supportsEventType`和`supportsSourceType`方法判断是否都为`true`否则通过`GenericApplicationListenerAdapter`适配器进行包装转化后判断

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