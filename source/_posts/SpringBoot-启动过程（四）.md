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
根据应用类型去实例化相应的容器对象，SpringBoot 使用`AnnotationConfigServletWebServerApplicationContext`，实例化会初始化`AnnotatedBeanDefinitionReader`和`ClassPathBeanDefinitionScanner` 
- AnnotatedBeanDefinitionReader 用于管理`BeanDefinition`，包含 `BeanDefinitionRegistry`注册管理、`BeanNameGenerator` beanName生成规则、`scopeMetadataResolver` Scope注解解析器
- ClassPathBeanDefinitionScanner 用与类扫描包含了匹配规则`**/*.class/`和过滤条件`@Component`（所以我们项目中常常使用的@Service、@Controller会被注册）、`scopeMetadataResolver` Scope注解解析器、`resourcePatternResolver`资源加载器等

``` java
// 根据类型对应Class利用反射创建实例 
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

// AnnotationConfigSelvetWebServerApplicationContext 构造方法 
public AnnotationConfigServletWebServerApplicationContext() {
    this.annotatedClasses = new LinkedHashSet();
    this.reader = new AnnotatedBeanDefinitionReader(this);
    this.scanner = new ClassPathBeanDefinitionScanner(this);
}

```

![ConfigurableApplicationContext 可配置上下文结构](/images/2020/05/19/beb83e10-99b4-11ea-bc19-85fa9aca2a18.png)

### SpringApplicationContext 准备
1. 上下文加载装配环境包含`AnnotatedBeanDefinitionReader`和`ClassPathBeanDefinitionScanner`
2. 配置上下文的 bean 生成器及 ResourceLoader 资源加载器，广播上下文准备事件并开启日志
3. 获取`DefaultListableBeanFactory`并单例注册`springApplicationArguments`、`springBootBanner`
4. 加载所有资源创建 `BeanDefinitionReader`读取资源根据资源类型分类解析
5. 广播上下文加载完毕事件

- XmlBeanDefinitionReader xml 配置文件读取加载 bean
- AnnotatedBeanDefinitionReader 注解方法读取
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
    // 广播上下文准备事件
    listeners.contextPrepared(context);
    if (this.logStartupInfo) {
        this.logStartupInfo(context.getParent() == null);
        this.logStartupProfileInfo(context);
    }
    // 可配置Bean 工厂 单例注册特殊bean
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
    // 加载所有资源
    Set<Object> sources = this.getAllSources();
    Assert.notEmpty(sources, "Sources must not be empty");
    this.load(context, sources.toArray(new Object[0]));
    // 广播上下文加载事件
    listeners.contextLoaded(context);
}
```

### SpringApplicationContext 刷新
``` java
public void refresh() throws BeansException, IllegalStateException {
    synchronized(this.startupShutdownMonitor) {
        # 准备刷新 更新状态、校验必要属性、设置监听器列表
        this.prepareRefresh();
        // 设置SerializationId 
        ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
        // 后置处理器设置注册各类环境配置Bean
        this.prepareBeanFactory(beanFactory);
        try {
            // 后置处理器空方法用于第三方拓展
            this.postProcessBeanFactory(beanFactory);
            this.invokeBeanFactoryPostProcessors(beanFactory);
            this.registerBeanPostProcessors(beanFactory);
            this.initMessageSource();
            this.initApplicationEventMulticaster();
            this.onRefresh();
            this.registerListeners();
            this.finishBeanFactoryInitialization(beanFactory);
            this.finishRefresh();
        } catch (BeansException var9) {
            if (this.logger.isWarnEnabled()) {
                this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + var9);
            }

            this.destroyBeans();
            this.cancelRefresh(var9);
            throw var9;
        } finally {
            this.resetCommonCaches();
        }

    }
}
```

#### prepareRefresh 准备刷新上下文
1. 设置上下文状态记录跟踪日志
2. 初始化配置资源并校验环境必要属性
3. 新增监听器列表 `earlyApplicationListeners`
``` java 
protected void prepareRefresh() {
    this.startupDate = System.currentTimeMillis();
    this.closed.set(false);
    this.active.set(true);
    if (this.logger.isDebugEnabled()) {
    if (this.logger.isTraceEnabled()) {
        this.logger.trace("Refreshing " + this);
    } else {
        this.logger.debug("Refreshing " + this.getDisplayName());
    }
    }

    this.initPropertySources();
    this.getEnvironment().validateRequiredProperties();
    if (this.earlyApplicationListeners == null) {
    this.earlyApplicationListeners = new LinkedHashSet(this.applicationListeners);
    } else {
    this.applicationListeners.clear();
    this.applicationListeners.addAll(this.earlyApplicationListeners);
    }

    this.earlyApplicationEvents = new LinkedHashSet();
}
```

#### prepareBeanFactory 准备 BeanFactory
1. 加载 `ClassLoader`、新增 `StandardBeanExpressionResolver` 标准的`Bean`表达式解析器、`ResourceEditorRegistrar` 属性编辑注册器
2. 添加 `ApplicationContextAwareProcessor` 后置处理器用于处理 Bean 初始化前后操作，忽略依赖接口列表新增`EnvironmentAware`、`ApplicationEventPushlisherAware`、`ResouceLoaderAware`、`EmbedderValueResolverAware`、`MessageSourceAware`、`ApplicationContextAware`接口，由于`ApplicationContextAwareProcesser`已实现以上功能
3. 新增`ApplicationListenerDetector` 监听器检测器用于内置 Bean 发布事件
4. 注册单例 Bean `environment`、`systemProperties`、`systemEnvironment`
``` java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // 设置 ClassLoader 用于加载 Bean 
    beanFactory.setBeanClassLoader(this.getClassLoader());
    // 设置 StandardBeanExpressionResolver（标准 Bean EL 表达式解析器） 用于解析#{}
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    // 新增 属性编辑注册器 
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, this.getEnvironment()));
    // 新增 Bean 后置处理器
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    // 忽略以下接口类依赖
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));
    if (beanFactory.containsBean("loadTimeWeaver")) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }

    if (!beanFactory.containsLocalBean("environment")) {
        beanFactory.registerSingleton("environment", this.getEnvironment());
    }

    if (!beanFactory.containsLocalBean("systemProperties")) {
        beanFactory.registerSingleton("systemProperties", this.getEnvironment().getSystemProperties());
    }

    if (!beanFactory.containsLocalBean("systemEnvironment")) {
        beanFactory.registerSingleton("systemEnvironment", this.getEnvironment().getSystemEnvironment());
    }

}
```

#### postProcessBeanFactory 后置处理器
根据第三方组件拓展新增后置处理器忽略接口依赖
``` java
// ServletWebServerApplicationContext 提供的方法 
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    beanFactory.addBeanPostProcessor(new WebApplicationContextServletContextAwareProcessor(this));
    beanFactory.ignoreDependencyInterface(ServletContextAware.class);
    this.registerWebApplicationScopes();
}


protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    super.postProcessBeanFactory(beanFactory);
    if (this.basePackages != null && this.basePackages.length > 0) {
        this.scanner.scan(this.basePackages);
    }

    if (!this.annotatedClasses.isEmpty()) {
        this.reader.register(ClassUtils.toClassArray(this.annotatedClasses));
    }
}
```

#### invokeBeanFactoryPostProcessors 调用 Bean 工厂后置处理器
1. 迭代上下文中`beanFactoryPostProcessors` 后置处理器集合，如果是`BeanDefinitionRegistryPostProcessor` 则调用执行`postProcessBeanDefinitionRegistry`否则仅收集与`regularPostProcessors`
2. 循环类型为`BeanDefinitionRegistryPostProcessor`的处理器名字集合判断
``` java
public static void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {
    Set<String> processedBeans = new HashSet();
    ArrayList regularPostProcessors;
    ArrayList registryProcessors;
    int var9;
    ArrayList currentRegistryProcessors;
    String[] postProcessorNames;
    if (beanFactory instanceof BeanDefinitionRegistry) {
        BeanDefinitionRegistry registry = (BeanDefinitionRegistry)beanFactory;
        regularPostProcessors = new ArrayList();
        registryProcessors = new ArrayList();
        Iterator var6 = beanFactoryPostProcessors.iterator();
        // 迭代判断类型分配收集和执行 postProcessBeanDefinitionRegistry
        while(var6.hasNext()) {
            BeanFactoryPostProcessor postProcessor = (BeanFactoryPostProcessor)var6.next();
            if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
                BeanDefinitionRegistryPostProcessor registryProcessor = (BeanDefinitionRegistryPostProcessor)postProcessor;
                registryProcessor.postProcessBeanDefinitionRegistry(registry);
                registryProcessors.add(registryProcessor);
            } else {
                regularPostProcessors.add(postProcessor);
            }
        }

        currentRegistryProcessors = new ArrayList();
        postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        String[] var16 = postProcessorNames;
        var9 = postProcessorNames.length;

        int var10;
        String ppName;
        // 
        for(var10 = 0; var10 < var9; ++var10) {
            ppName = var16[var10];
            if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
            }
        }

        sortPostProcessors(currentRegistryProcessors, beanFactory);
        registryProcessors.addAll(currentRegistryProcessors);
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
        currentRegistryProcessors.clear();
        postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        var16 = postProcessorNames;
        var9 = postProcessorNames.length;

        for(var10 = 0; var10 < var9; ++var10) {
            ppName = var16[var10];
            if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
            }
        }

        sortPostProcessors(currentRegistryProcessors, beanFactory);
        registryProcessors.addAll(currentRegistryProcessors);
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
        currentRegistryProcessors.clear();
        boolean reiterate = true;

        while(reiterate) {
            reiterate = false;
            postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
            String[] var19 = postProcessorNames;
            var10 = postProcessorNames.length;

            for(int var26 = 0; var26 < var10; ++var26) {
                String ppName = var19[var26];
                if (!processedBeans.contains(ppName)) {
                    currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                    processedBeans.add(ppName);
                    reiterate = true;
                }
            }

            sortPostProcessors(currentRegistryProcessors, beanFactory);
            registryProcessors.addAll(currentRegistryProcessors);
            invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
            currentRegistryProcessors.clear();
        }

        invokeBeanFactoryPostProcessors((Collection)registryProcessors, (ConfigurableListableBeanFactory)beanFactory);
        invokeBeanFactoryPostProcessors((Collection)regularPostProcessors, (ConfigurableListableBeanFactory)beanFactory);
    } else {
        invokeBeanFactoryPostProcessors((Collection)beanFactoryPostProcessors, (ConfigurableListableBeanFactory)beanFactory);
    }

    String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);
    regularPostProcessors = new ArrayList();
    registryProcessors = new ArrayList();
    currentRegistryProcessors = new ArrayList();
    postProcessorNames = postProcessorNames;
    int var20 = postProcessorNames.length;

    String ppName;
    for(var9 = 0; var9 < var20; ++var9) {
        ppName = postProcessorNames[var9];
        if (!processedBeans.contains(ppName)) {
            if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                regularPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
            } else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
                registryProcessors.add(ppName);
            } else {
                currentRegistryProcessors.add(ppName);
            }
        }
    }

    sortPostProcessors(regularPostProcessors, beanFactory);
    invokeBeanFactoryPostProcessors((Collection)regularPostProcessors, (ConfigurableListableBeanFactory)beanFactory);
    List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList(registryProcessors.size());
    Iterator var21 = registryProcessors.iterator();

    while(var21.hasNext()) {
        String postProcessorName = (String)var21.next();
        orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
    }

    sortPostProcessors(orderedPostProcessors, beanFactory);
    invokeBeanFactoryPostProcessors((Collection)orderedPostProcessors, (ConfigurableListableBeanFactory)beanFactory);
    List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList(currentRegistryProcessors.size());
    Iterator var24 = currentRegistryProcessors.iterator();

    while(var24.hasNext()) {
        ppName = (String)var24.next();
        nonOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
    }

    invokeBeanFactoryPostProcessors((Collection)nonOrderedPostProcessors, (ConfigurableListableBeanFactory)beanFactory);
    beanFactory.clearMetadataCache();
}
```
