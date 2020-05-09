---
title: Springboot Bean 加载过程源码解析
tags:
  - Spring
categories:
  - Spring
toc: false
date: 2020-04-14 14:14:36
---

> 面试问道Spring Bean 加载过程一脸懵逼哦

## SpringBoot 启动
SpringBoot 以`main`方法来启动项目构造方法，`SpringApplication` 构造方法用于初始化各种参数

``` java
package com;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

``` java
// SpringApplication 构造方法
public SpringApplication(Object... sources) {
    this.sources = new LinkedHashSet();
    // 控制台打印
    this.bannerMode = Mode.CONSOLE;
    // 开启日志
    this.logStartupInfo = true;
    // 开启命令行参数 以 -- 开头
    this.addCommandLineProperties = true;
    // 开启转化 service
    this.addConversionService = true;
    // 开启无头
    this.headless = true;
    // 开启注册关闭 Hook
    this.registerShutdownHook = true;
    // 额外 profiles
    this.additionalProfiles = new HashSet();
    // 自定义环境
    this.isCustomEnvironment = false;
    // 资源装载类
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    // 主要参数 args[]
    this.primarySources = new LinkedHashSet(Arrays.asList(primarySources));
    // 判断应用类型 WebApplicationType 枚举 ReActive、Servlet、None 
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    // 加载初始化器和监听器 
    this.setInitializers(this.getSpringFactoriesInstances(ApplicationContextInitializer.class));
    this.setListeners(this.getSpringFactoriesInstances(ApplicationListener.class));
    // 判断 main 方法入口类
    this.mainApplicationClass = this.deduceMainApplicationClass();
}
```

 监听器和初始化器通过遍历解析所有依赖包中`META-INF`文件夹下`spring.factories`得到需要加载的各类组件然后通过反射实例化，很多第三方组件`starter`就是编写`spring.factories`来实现自动装配，`run` 方法开始进行各种环境的配置准备刷新

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

#### SpringApplicationRunListener 监听器初始化

`SpringApplicationRunListeners listeners = this.getRunListeners(args)`这行代码通过`spring.properties`获取`EventPublishingRunListener`实例化用于事件广播，构造函数将所有监听器加入广播之中

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

#### ConfigurableEnvironment 初始化