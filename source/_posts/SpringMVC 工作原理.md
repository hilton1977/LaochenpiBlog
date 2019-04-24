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
通过继承`FrameworkServlet`抽象类复写`onRefresh`方法，根据上下文`ApplicationContext`对象执行`initStrategies`初始化策略。

### HandlerMapping
根据请求的`Url`找到对应的`Handler`，即我们项目中平时所谓的`Controller`，我们可以通过配置文件、注解方式、实现接口方式。
![HandlerMapping 类图](/images/HandlerMapping.png)
- **BeanNameUrlHandlerMapping** 通过`bean`名称或别名，必须已`/`开头和方法类必须实现`Controller`接口。
``` java
<bean name="/beanNameUrlController" class="com.test.web.MessageControl"/>

public class MessageControl implements Controller {
    public ModelAndView handleRequest(HttpServletRequest request,
           HttpServletResponseresponse) throws Exception {
       ModelAndView mav = new ModelAndView("index");
       return mav;
    }
}
```

- **SimpleUrlHandlerMapping** 
``` java

<bean class="org.springframework.web.servlet.handler.SimpleUrlHandlerMapping">
    <property name="mappings">
        <props>
            <prop key="/test.do">testController</prop>
        </props>
     </property>
</bean>
<bean id="testController" class="com.test.controller.SimpleUrlHandlerMappingController" />
 
 
public class SimpleUrlHandlerMappingController implements Controller {
    protected final Log logger = LogFactory.getLog(this.getClass());
    //@Override
    public ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws Exception {
        logger.info("run here");
        return null;
}
```

- **RequestMappingHandlerMapping** 项目中最常用的一种配置方法，配合`@Controller`和`@RequestMapping`一起使用。
``` java

<!-- 注册HandlerMapper、HandlerAdapter两个映射类 -->
<mvc:annotation-driven />
 
<!-- 访问静态资源 -->
<mvc:default-servlet-handler />
 
<!-- 配置扫描的包 -->
<context:component-scan base-package="com.test.*" />
 
@RestController
@RequestMapping("/hello")
public class HelloController {
    protected final Log logger = LogFactory.getLog(this.getClass());
 
    @RequestMapping("/index")
    public String index(){
        return "test";
}
```