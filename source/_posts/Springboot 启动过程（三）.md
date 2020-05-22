---
title: SpringBoot 启动源码解析（三）之配置环境
tags:
  - Spring
categories:
  - Spring
toc: false
date: 2020-05-09 10:14:36
---

![](/images/spring.jpg)
> 继监听器初始化完毕接来下开始可配置环境预处理

### ApplicationArguments 默认参数初始化
初始化`DefaultApplicationArguments`默认应用参数，通过`SimpleCommandLineArgsParser`对额外参数数组循环读取以`--`开头的命令行分割形成键值对封装为`SimpleCommandLinePropertySource`对象
``` java
public CommandLineArgs parse(String... args) {
    CommandLineArgs commandLineArgs = new CommandLineArgs();
    String[] var3 = args;
    int var4 = args.length;
    
    for(int var5 = 0; var5 < var4; ++var5) {
        String arg = var3[var5];
        if (arg.startsWith("--")) {
            String optionText = arg.substring(2);
            String optionValue = null;
            int indexOfEqualsSign = optionText.indexOf(61);
            String optionName;
            if (indexOfEqualsSign > -1) {
                optionName = optionText.substring(0, indexOfEqualsSign);
                optionValue = optionText.substring(indexOfEqualsSign + 1);
            } else {
                optionName = optionText;
            }

            if (optionName.isEmpty()) {
                throw new IllegalArgumentException("Invalid argument syntax: " + arg);
            }

            commandLineArgs.addOptionArg(optionName, optionValue);
        } else {
            commandLineArgs.addNonOptionArg(arg);
        }
    }
    return commandLineArgs;
}
```

### ConfigurableEnvironment 准备
默认参数初始化完毕后开始配置环境准备阶段
``` java
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments) {
    // 根据当前应用类型实例化环境
    ConfigurableEnvironment environment = this.getOrCreateEnvironment();
    // 命令行额外参数添加
    this.configureEnvironment((ConfigurableEnvironment)environment, applicationArguments.getSourceArgs());
    // SpringConfigurationPropertySources 保存之前 MutablePropertySources 数据
    ConfigurationPropertySources.attach((Environment)environment);
    // 监听器广播配置环境准备事件
    listeners.environmentPrepared((ConfigurableEnvironment)environment);
    // 绑定 Spring 默认的配置文件中信息
    this.bindToSpringApplication((ConfigurableEnvironment)environment);
    // 是否为自定义容器 否则进行容器转化
    if (!this.isCustomEnvironment) {
        environment = (new EnvironmentConverter(this.getClassLoader())).convertEnvironmentIfNecessary((ConfigurableEnvironment)environment, this.deduceEnvironmentClass());
    }

    ConfigurationPropertySources.attach((Environment)environment);
    return (ConfigurableEnvironment)environment;
}
```

#### getOrCreateEnvironment 
根据当前应用类型实例化不同的环境，所有环境对象都继承`StandardEnvironment`，默认配置环境`MutablePropertySources`对象里`propertySourceList`会新增以下`PropertySource`
``` java
private ConfigurableEnvironment getOrCreateEnvironment() {
    if (this.environment != null) {
        return this.environment;
    } else {
        switch(this.webApplicationType) {
        case SERVLET:
            return new StandardServletEnvironment();
        case REACTIVE:
            return new StandardReactiveWebEnvironment();
        default:
            return new StandardEnvironment();
        }
    }
}
```
####  configureEnvironment 配置环境
根据之前实例化的配置环境加载`propertySource`、`profiles`
``` java
protected void configureEnvironment(ConfigurableEnvironment environment, String[] args) {
    if (this.addConversionService) {
        ConversionService conversionService = ApplicationConversionService.getSharedInstance();
        environment.setConversionService((ConfigurableConversionService)conversionService);
    }
    // 命令行加入属性资源
    this.configurePropertySources(environment, args);
    this.configureProfiles(environment, args);
}
```
实例化`ConversionService`并加载到环境中，内部方法使用了**双重验证同步锁**单例模式
``` java
public static ConversionService getSharedInstance() {
    ApplicationConversionService sharedInstance = sharedInstance;
    if (sharedInstance == null) {
        Class var1 = ApplicationConversionService.class;
        # 添加同步锁
        synchronized(ApplicationConversionService.class) {
            sharedInstance = sharedInstance;
            if (sharedInstance == null) {
                sharedInstance = new ApplicationConversionService();
                sharedInstance = sharedInstance;
            }
        }
    }

    return sharedInstance;
}
```
将之前的命令行配置环境新增或替换配置环境的`MutablePropertySources`，装配面板额外添加 `configureProfiles` ，在实际开发中用于测试生产配置分离，不同情况下激活不同的`profile`
``` java
protected void configurePropertySources(ConfigurableEnvironment environment, String[] args) {
    MutablePropertySources sources = environment.getPropertySources();
    if (this.defaultProperties != null && !this.defaultProperties.isEmpty()) {
        sources.addLast(new MapPropertySource("defaultProperties", this.defaultProperties));
    }
    // 命令行配置是否开启同时命令行数组大于0
    if (this.addCommandLineProperties && args.length > 0) {
        String name = "commandLineArgs";
        // 如果有则进行替换否则新增
        if (sources.contains(name)) {
            PropertySource<?> source = sources.get(name);
            CompositePropertySource composite = new CompositePropertySource(name);
            composite.addPropertySource(new SimpleCommandLinePropertySource("springApplicationCommandLineArgs", args));
            composite.addPropertySource(source);
            sources.replace(name, composite);
        } else {
            sources.addFirst(new SimpleCommandLinePropertySource(args));
        }
    }
}

protected void configureProfiles(ConfigurableEnvironment environment, String[] args) {
    Set<String> profiles = new LinkedHashSet(this.additionalProfiles);
    profiles.addAll(Arrays.asList(environment.getActiveProfiles()));
    environment.setActiveProfiles(StringUtils.toStringArray(profiles));
}
```
接下来将`ConfigurationPropertySourcesPropertySource`加入`MutablePropertySources`，然后通过监听器广播发送`ApplicationEnvironmentPreparedEvent`应用环境准备事件，最后返回最终的容器中配属性资源包含以下

- **PropertiesPropertySource {name='systemProperties'}** ：保存当前系统配置情况的键值对信息（缺省信息）
- **SystemEnvironmentPropertySource {name='systemEnvironment'}** ： 对象保存当前系统环境变量键值对信息（缺省信息）
- **StubPropertySource {name='servletContextInitParams'}** ：保存上下文初始化参数 （Servlet 特有）
- **StubPropertySource {name='servletConfigInitParams'}** ：保存设置初始化参数（Servlet 特有）
- **JndiPropertySource {name='jndiProperties'}** ：保存Jndi相关配置（Servlet 特有）
- **SimpleCommandLinePropertySource {name='commandLineArgs'}** ：保存命令行额外参数
- **ConfigurationPropertySourcesPropertySource {name='configurationProperties'}** ：保存 MutablePropertySources 信息包含所有的配置信息
- **OriginTrackedMapPropertySource {name='applicationConfig: [classpath:/application.yml]'}** ：保存`spring`默认配置文件`properties`或`yml`中键值对信息

