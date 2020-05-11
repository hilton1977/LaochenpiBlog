---
title: SpringBoot 启动源码解析（二）
tags:
  - Spring
categories:
  - Spring
toc: false
date: 2020-05-11 14:14:36
---
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
            //准备 刷新 配置上下文
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

#### SpringApplicationRunListeners 事件监听器推送
SpringApplicationRunListeners 实例化最重要的就是`EventPublishingRunListener`用于事件广播，构造函数将所有实例化过监听器加入`SimpleApplicationEventMulticaster`广播之中

``` java
public EventPublishingRunListener(SpringApplication application, String[] args) {
    this.application = application;
    this.args = args;
    this.initialMulticaster = new SimpleApplicationEventMulticaster();
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
        this.retrieverCache.clear();
    }
}
```

每次事件通知调用`multicastEvent`方法，遍历调用监听器`onApplicationEvent`方法，如果有线程池则异步调用否则同步调用，`this.getApplicationListeners(event, type)`过滤监听器集合，通过调用监听器`supportsEventType`、`supportsSourceType`方法来判断是否支持当前事件

``` java
public void multicastEvent(ApplicationEvent event, @Nullable ResolvableType eventType) {
    ResolvableType type = eventType != null ? eventType : this.resolveDefaultEventType(event);
    Iterator var4 = this.getApplicationListeners(event, type).iterator();

    while(var4.hasNext()) {
        ApplicationListener<?> listener = (ApplicationListener)var4.next();
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