---
title: SpringMVC 运行原理
date: 2019-04-22 14:56:12
categories: [Java]
tags:
	 - Spring
---
![](/images/spring.jpg)
> 面试经常会问到 SpringMVC 运行原理真是一脸懵逼，太细节的都不记得了就知道撸代码，现在记录并复习下。

### SpringMVC 工作原理
![SpringMVC 工作流程](/images/springmvc.jpg)
1. 用户通过浏览器访问项目，请求会发送到`DispatcherServlet`前端控制器
2. 通过`HandlerMapping`找到对应的`Controller`对应注解`@RequestMapping`的值
3. 根据`Controller`方法通过`HandlerAdapter`处理适配器查找`handler`
4. `HandlerAdapter`执行适配的`handler`开始正式的业务逻辑执行
5. 处理完业务会返回`ModleAndView`，`Model`为返回数据，`View`为返回的视图
6. 根据`View`执行`ViewResolver`找到对应逻辑页面
7. `DispatcherSerlvet`会把`Model`传给`View`进行页面渲染

### DispatcherServlet
前端控制器是 SpringMVC 的核心是所有请求的入口函数，处理请求、转发请求、处理结果、返回结果，默认初始化一些`HandlerAdapter`、`HandMapping`、`HandlerExceptionResolver`等组件
``` java
static {
        try {
            ClassPathResource resource = new ClassPathResource("DispatcherServlet.properties", DispatcherServlet.class);
            defaultStrategies = PropertiesLoaderUtils.loadProperties(resource);
        } catch (IOException var1) {
            throw new IllegalStateException("Could not load 'DispatcherServlet.properties': " + var1.getMessage());
        }
    }
```
读取`DispatcherServlet.properties`初始化使用的策略
``` java
protected void onRefresh(ApplicationContext context) {
    this.initStrategies(context);
}

protected void initStrategies(ApplicationContext context) {
    this.initMultipartResolver(context);
    this.initLocaleResolver(context);
    this.initThemeResolver(context);
    this.initHandlerMappings(context);
    this.initHandlerAdapters(context);
    this.initHandlerExceptionResolvers(context);
    this.initRequestToViewNameTranslator(context);
    this.initViewResolvers(context);
    this.initFlashMapManager(context);
}
```
通过继承`FrameworkServlet`抽象类复写`onRefresh`方法，根据上下文`ApplicationContext`对象执行`initStrategies`初始化策略，

### HandlerMapping
根据请求的`Url`找到对应的`Handler`，即我们项目中平时所谓的`Controller`，我们可以通过配置文件、注解方式、实现接口方式。
![HandlerMapping 类图](/images/HandlerMapping.png)