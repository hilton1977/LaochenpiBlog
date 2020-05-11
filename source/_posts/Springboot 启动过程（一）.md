---
title: SpringBoot 启动源码解析（一）
tags:
  - Spring
categories:
  - Spring
toc: false
date: 2020-04-14 14:14:36
---
![](/images/spring.jpg)
> SpringBoot 启动源码解析（一） 看看 SpringApplication 实例化过程中就干了啥

## SpringBoot 实例化
SpringBoot 以`main`方法来启动项目构造方法，调用静态方法`run`内部在实例化 SpringApplication 从参数来看，我们可以启动多个 SpringApplication
``` java

public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
        return run(new Class[]{primarySource}, args);
}

public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
        return (new SpringApplication(primarySources)).run(args);
}

```

SpringApplication 的构造方法用于初始化各种默认参数监听器、初始化器、WebApplicationType、命令行参数等

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

#### WebApplicationType.deduceFromClasspath() 应用类型判断
用于判断当前项目是否为`Servlet`环境，看代码可知通过不存在`DispatcherHandler`（分发处理器）、`DispatcherServlet`（分发中心）、`ServletContainer`（Servlet 容器）3个类来判断响应式应用，通过存在`Servlet`、`ConfigurableWebApplicationContext`同时存在来判断`Servlet`环境

``` java
static WebApplicationType deduceFromClasspath() {
    if (ClassUtils.isPresent("org.springframework.web.reactive.DispatcherHandler", (ClassLoader)null) && !ClassUtils.isPresent("org.springframework.web.servlet.DispatcherServlet", (ClassLoader)null) && !ClassUtils.isPresent("org.glassfish.jersey.servlet.ServletContainer", (ClassLoader)null)) {
        return REACTIVE;
    } else {
        String[] var0 = SERVLET_INDICATOR_CLASSES;
        int var1 = var0.length;

        for(int var2 = 0; var2 < var1; ++var2) {
            String className = var0[var2];
            if (!ClassUtils.isPresent(className, (ClassLoader)null)) {
                return NONE;
            }
        }
        return SERVLET;
    }
}
```

#### setInitializers、setListeners 初始化监听器和初始化器

监听器和初始化器都用调用`getSpringFactoriesInstances`来进行实例化
``` java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = this.getClassLoader();
    Set<String> names = new LinkedHashSet(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    List<T> instances = this.createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
```

通过`SpringFactoriesLoader.loadSpringFactories`遍历获取所有依赖包中`META-INF/spring.factories`文件解析出键值对`Map<String, List<String>>`集合，很多第三方组件`starter`就是编写`spring.factories`来实现自动装配功能，这里使用了静态缓存`SpringFactoriesLoader.cache`只需一次解析下次直接读取缓存

``` java
private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
    MultiValueMap<String, String> result = (MultiValueMap)cache.get(classLoader);
    if (result != null) {
            return result;
    } else {
        try {
            Enumeration<URL> urls = classLoader != null ? classLoader.getResources("META-INF/spring.factories") : ClassLoader.getSystemResources("META-INF/spring.factories");
            LinkedMultiValueMap result = new LinkedMultiValueMap();

            while(urls.hasMoreElements()) {
                URL url = (URL)urls.nextElement();
                UrlResource resource = new UrlResource(url);
                Properties properties = PropertiesLoaderUtils.loadProperties(resource);
                Iterator var6 = properties.entrySet().iterator();

                while(var6.hasNext()) {
                    Entry<?, ?> entry = (Entry)var6.next();
                    String factoryClassName = ((String)entry.getKey()).trim();
                    String[] var9 = StringUtils.commaDelimitedListToStringArray((String)entry.getValue());
                    int var10 = var9.length;

                    for(int var11 = 0; var11 < var10; ++var11) {
                        String factoryName = var9[var11];
                        result.add(factoryClassName, factoryName.trim());
                    }
                }
            }

                cache.put(classLoader, result);
                return result;
            } catch (IOException var13) {
                throw new IllegalArgumentException("Unable to load factories from location [META-INF/spring.factories]", var13);
            }
        }
    }
```

通过`createSpringFactoriesInstances`利用类反射特性来实例化之前获取`Class`集合

``` java
private <T> List<T> createSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, ClassLoader classLoader, Object[] args, Set<String> names) {
    List<T> instances = new ArrayList(names.size());
    Iterator var7 = names.iterator();

    while(var7.hasNext()) {
        String name = (String)var7.next();

        try {
            Class<?> instanceClass = ClassUtils.forName(name, classLoader);
            Assert.isAssignable(type, instanceClass);
            Constructor<?> constructor = instanceClass.getDeclaredConstructor(parameterTypes);
            T instance = BeanUtils.instantiateClass(constructor, args);
            instances.add(instance);
        } catch (Throwable var12) {
            throw new IllegalArgumentException("Cannot instantiate " + type + " : " + name, var12);
        }
    }
        return instances;
    }
```

### deduceMainApplicationClass 判断 main Class
获取当前线程堆栈跟踪元素然后遍历调用链判断每个方法名称是否为`main`获取入口类
``` java
private Class<?> deduceMainApplicationClass() {
    try {
        StackTraceElement[] stackTrace = (new RuntimeException()).getStackTrace();
        StackTraceElement[] var2 = stackTrace;
        int var3 = stackTrace.length;

        for(int var4 = 0; var4 < var3; ++var4) {
            StackTraceElement stackTraceElement = var2[var4];
            if ("main".equals(stackTraceElement.getMethodName())) {
                return Class.forName(stackTraceElement.getClassName());
            }
        }
    } catch (ClassNotFoundException var6) {
    }

    return null;
}   
```
