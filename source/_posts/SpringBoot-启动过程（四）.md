---
title: SpringBoot 启动源码解析（四）之容器
tags:
  - Spring
categories:
  - Spring
toc: false
date: 2020-05-20 10:14:36
---

![](/images/spring.jpg)
> 当配置环境初始化完毕后下一步容器创建、准备、刷新、刷新后处理

### SpringApplicationContext 容器创建
于之前一样创建容器也需要根据应用类型去实例化相应的容器对象，SpringBoot 可使用`AnnotationConfigServletWebServerApplicationContext`注解方式装配 Bean，SpringMVC 可使用`XmlServletWebServerApplicationContext`通过XML配置文件来装配 Bean 

``` java
protected ConfigurableApplicationContext createApplicationContext() {
    Class<?> contextClass = this.applicationContextClass;
    if (contextClass == null) {
        try {
            switch(this.webApplicationType) {
            case SERVLET:
                contextClass = Class.forName("org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext");
                break;
            case REACTIVE:
                contextClass = Class.forName("org.springframework.boot.web.reactive.context.AnnotationConfigReactiveWebServerApplicationContext");
                break;
            default:
                contextClass = Class.forName("org.springframework.context.annotation.AnnotationConfigApplicationContext");
            }
        } catch (ClassNotFoundException var3) {
            throw new IllegalStateException("Unable create a default ApplicationContext, please specify an ApplicationContextClass", var3);
        }
    }

    return (ConfigurableApplicationContext)BeanUtils.instantiateClass(contextClass);
}
```
![ConfigurableApplicationContext 可配置上下文结构](/images/2020/05/19/beb83e10-99b4-11ea-bc19-85fa9aca2a18.png)

### SpringApplicationContext 准备
加载之前的配置环境，配置上下文的 bean 生成器及 ResourceLoader 资源加载器，`ConfigurableListableBeanFactory`单例注册特殊`Bean`，创建 BeanDefinitionLoader 加载资源这里有我们熟知的各种加载方法，然后根据已有 Sources 进行判断用那种读取器

- XmlBeanDefinitionReader xml 配置文件读取加载 bean
- AnnotatedBeanDefinitionReader Springboot 最常用方法使用注解方法 @Service @Controller 等
- ClassPathBeanDefinitionScanner 类路径扫描器
- GroovyBeanDefinitionReader groovy 方法加载 bean
``` java
private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment, SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
    // 装载可配置环境 
    context.setEnvironment(environment);
    // 配置上下文的 bean 生成器及资源加载器
    this.postProcessApplicationContext(context);
    // 循环遍历所有初始化器
    this.applyInitializers(context);
    // 监听器广播上下文准备事件
    listeners.contextPrepared(context);
    if (this.logStartupInfo) {
        this.logStartupInfo(context.getParent() == null);
        this.logStartupProfileInfo(context);
    }
    // 可配置Bean 工厂 注册对象
    ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
    beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
    if (printedBanner != null) {
        beanFactory.registerSingleton("springBootBanner", printedBanner);
    }

    if (beanFactory instanceof DefaultListableBeanFactory) {
        ((DefaultListableBeanFactory)beanFactory).setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
    }

    if (this.lazyInitialization) {
        context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
    }

    Set<Object> sources = this.getAllSources();
    Assert.notEmpty(sources, "Sources must not be empty");
    this.load(context, sources.toArray(new Object[0]));
    // 监听器广播上下文加载事件
    listeners.contextLoaded(context);
}
```

### SpringApplicationContext 刷新
