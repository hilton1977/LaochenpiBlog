---
title: Spring 动态注入 Bean 以及修改
tags:
  - Java
categories:
- Spring
toc: false
date: 2020-11-18 14:53:09
---
### ImportBeanDefinitionRegistrar 类
实现`ImportBeanDefinitionRegistrar`类进行动态注册，**必须配合@Import注解进行使用**，`Import`注解用于导入一些未被将其交给Spring容器去管理，例如`@MapperScan`注册对应的`Mapper`类就通过`@Import`导入`MapperScannerRegistrar`类实现`ImportBeanDefinitionRegistrar`接口进行自定义Bean注入

``` java

/**
 * @author lenovo
 */
public class MyImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        BeanDefinitionBuilder definitionBuilder = BeanDefinitionBuilder.genericBeanDefinition(TestDemo.class);
        definitionBuilder.addPropertyValue("name", "mytest");
        registry.registerBeanDefinition("mytest",definitionBuilder.getBeanDefinition());
    }
}
```
以上代码中`registry.registerBeanDefinition(beanName, builder.getBeanDefinition());`将`MapperScannerConfigurer` 注册至IOC中用于后续的Mapper Bean 的扫描，`BeanDefinitionBuilder`用于对Bean定义信息,Mybatis 就是通过这样的方法把扫描指定路径的Mapper接口类并创建代理类交予IOC进行管理

### BeanDefinitionRegistryPostProcessor 后置处理器

实现`BeanDefinitionRegistryPostProcessor`接口可以对已有`BeanDefinition`进行修改也可以根据业务需求进行动态注册自己的`Bean`、

- `BeanDefinitionPostProcessor` 执行在容器加载所有`bean`完毕之后实例化之前
- 可以配置多个`BeanDefinitionRegistryPostProcessor`在容器刷新时遍历所有的后置处理器

``` java
@Component
public class MyBeanDefinitionRegistryPostProcessor implements BeanDefinitionRegistryPostProcessor {
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
      BeanDefinition beanDefinition =  registry.getBeanDefinition("mytest");
        Arrays.stream(beanDefinition.getPropertyValues().getPropertyValues())
                .forEach(propertyValue -> {
                    propertyValue.setConvertedValue("change");
                });
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory configurableListableBeanFactory) throws BeansException {

    }
}
```

> ImportBeanDefinitionRegistrar 的执行在 BeanDefinitionPostProcessor 之前且必须配合@Import进行
