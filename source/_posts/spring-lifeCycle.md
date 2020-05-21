---
title: Spring之IOC(四)Bean的生命周期
date: 2020-05-21 22:14:12
tags: Spring
categories: Spring
---

今天回顾Spring的相关知识，我觉得还需要对Bean的生命周期总结一下。

![](spring-lifeCycle/015F958D.jpg)

<!-- more -->

## 前情回顾

我们讨论bean的生命周期，必然是从bean实例化开始。但是我们先简单回顾一下，在实例化之前需要干些什么事情：

1. xml文件或者注解由统一资源定位器ResourceLoader进行加载为统一的资源抽象Resource，再由BeanDefinitionReader进行资源解析为BeanDefinition，然后由BeanDefinitionRegistry将BeanDefinition进行注册，其实就是将BeanDefinition注入到一个HashMap容器中。如果在bean实例化之前有BeanFactoryPostProcessor，则会对注册到容器中的BeanDefinition进行修改。
2. 当我们显式或者隐式的方式调用BeanFactory的getBean()方法时，才会触发该类的实例化方法。当调用getBean()方法时，会调用deGetBean()方法，在该方法中，会先getSingleton()从三级缓存中取，如果取不到bean，则会调用另外一个getSinleton()重载方法，然后回调createBean()方法，在该方法中通过doCreateBean()来进行bean实例化工作。

接下来就到了生命周期的范畴了。

## Bean的生命周期

### bean实例化

在doCreateBean()中进行实例化工作，首先通过createBeanInstance()来创建一个BeanWrapper对象，在该方法中，会使用Supplier回调、工厂方法初始化、构造函数初始化等方式进行初始化，并且使用BeanWrapper进行包装。在实例化bean过程中，Spring采用“策略模式”来决定采用哪种方式来实例化bean，一般有反射和CGLIB动态字节码两种方式。

### 属性注入

这里有个小tip：在属性注入之前，为了避免循环依赖，在 bean 初始化完成前，就将创建 bean 实例的 ObjectFactory 放入工厂缓存（singletonFactories）。

随后调用populateBean()方法进行属性注入，该方法作用在于将BeanDefinition中的属性值赋值给BeanWrapper实例对象。

### 初始化

在完成Bean对象实例化并且设置完相关属性和依赖后，会调用Bean的初始化方法initializeBean()。

在该方法中，初始化第一个阶段检查当前Bean对象是否实现了BeanNameAware、BeanClassLoaderAware、BeanFactoryAware等接口，即激活Aware。

第二步，则是BeanPostProcessor增强前置处理，它主要对Spring容器提供的Bean实例对象进行有效的扩展，允许Spring在初始化Bean阶段对其进行定制化修改。

第三步，检查和执行InitializingBean和init-method方法。判断当前bean是否实现了InitializingBean，如果实现了则调用afterPropertiesSet方法，进行初始化工作，同样再判断是否指定了init-method，如果指定了，则通过反射机制调用指定的init-method方法。

第四步，执行BeanPostProcess增强后置处理。AbstractAutoProxyCreator实现了BeanPostProcessor的postProcessAfterInitialization方法，这里会生成代理对象，这里就是AOP实现的位置。

最后注册必要的回调函数，该bean就可以供容器使用。

### 执行销毁方法

当完成调用后，如果是singleton类的bean，则会看当前bean是否实现了DisposableBean接口或者配置了destory-method属性，如果是的话，则会为该实例注册一个用于对象销毁的回调方法，以便在这些singleton类型的bean对象销毁之前执行销毁逻辑。

1. 对于BeanFactory容器而言，我们需要主动调用 destorySingletons()通知BeanFactory容器去执行相应的销毁方法。
2. 对于ApplicationContext容器而言，调用 registerShutdownHook方法。

### 总结

所以综上，bean的生命周期如图所示：

![](spring-lifeCycle/Cgq2xl6WvHqAdmt4AABGAn2eSiI631.png)

> 参考列表：
>
> 1. http://cmsblogs.com/?p=4034