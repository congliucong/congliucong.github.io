---
title: Spring之IOC(三)ApplicationContext的refresh()
date: 2020-05-08 16:14:52
tags: Spring
categories: Spring
---

上篇文章我们通过手动加载bean，来大概的走了一遍bean的加载整个流程。这篇文章我们从ApplicationContext的角度来分析一下这个过程。

## ApplicationContext

![](spring-refresh/15409479819725.jpg)
由ApplicationContext的类图我们可以看出来，ApplicationContext继承：

1. BeanFactory ：Spring 管理Bean的顶层接口。ApplicationContext继承BeanFactory的两个子类：HierarchicalBeanFctory 和 ListableBeanFactory。HierarchicalBeanFactory是一个具有层级关系的BeanFactory，拥有属性parentBeanFactory。ListableBeanFactory实现了枚举方法可以列举出当前的BeanFactory中所有bean对象而不必根据name一个一个的获取。
2. ApplicationEventPublisher :用于封装事件发布功能的接口，向事件监听器发送事件消息。
3. ResourceLoader ： Spring加载资源的顶层接口，用于从一个源加载资源文件。Application继承ResourceLoader的子类ResourcePatternResolver，该接口是将location解析为Resource对象的策略接口。
4. MessageSource : 解析message的策略接口，用来支撑国际化等功能。
5. EnvironmentCaple ： 用于获取Environment的接口。

## ApplicationContext.refresh()

refresh()方法恐怕是ApplicationContext最重要的方法。当new ApplicationContext()会执行refresh方法。作用是：刷新Spring应用的上下文。

```java
    public ClassPathXmlApplicationContext(
            String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
            throws BeansException {

        super(parent);
        setConfigLocations(configLocations);
        if (refresh) {
            refresh();
        }
    }
```

```java
    @Override
    public void refresh() throws BeansException, IllegalStateException {
        synchronized (this.startupShutdownMonitor) {
            // 1. 准备刷新上下文环境
            prepareRefresh();

            // 2. 创建并初始化BeanFactory
            ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

            // 3.填充BeanFactory功能。
            prepareBeanFactory(beanFactory);

            try {
                // 4. 提供子类覆盖的额外处理，即子类处理自定义的BeanFactoryPostProcess
                postProcessBeanFactory(beanFactory);

                // 5.激活各种BeanFactory处理器
                invokeBeanFactoryPostProcessors(beanFactory);

                // 6.注册拦截Bean创建的Bean处理器，即注册BeanPostProcessor
                registerBeanPostProcessors(beanFactory);

                // 7. 初始化上下文的资源文件，如国际化文件的处理
                initMessageSource();

                // 8. 初始化上下文事件广播器
                initApplicationEventMulticaster();

                // 9. 给子类扩展初始化其他Bean
                onRefresh();

                // 10.在所有Bean中查找listener bean，然后注册到广播器中
                registerListeners();

                // 11.初始化剩下的单例Bean(非延时加载的)
                finishBeanFactoryInitialization(beanFactory);

                // 12.完成刷新过程，通知声明周期处理器lifcycleProcessor刷新过程，同时发出contextRefreshEcent通知别人
                finishRefresh();
            }

            catch (BeansException ex) {
                if (logger.isWarnEnabled()) {
                    logger.warn("Exception encountered during context initialization - " +
                            "cancelling refresh attempt: " + ex);
                }

                // 如果发生异常，销毁已经创建的bean
                destroyBeans();

                // 重置容器激活标签
                cancelRefresh(ex);

                // Propagate exception to caller.
                throw ex;
            }

            finally {
                // Reset common introspection caches in Spring's core, since we
                // might not ever need metadata for singleton beans anymore...
                resetCommonCaches();
            }
        }
    }
```
一共12个方法，可不可怕？ 我们来一个一个大致过一遍。

### prepareRefresh()

这个方法是准备刷新上下文环境。主要对系统的环境变量或者系统属性进行准备和校验。比如重写initPropertySources方法，要求环境变量中必须设置某个值才能进行刷新上下文，否则不能运行，

```java
    protected void prepareRefresh() {
        // 设置启动日期
        this.startupDate = System.currentTimeMillis();
        //设置context当前状态
        this.closed.set(false);
        this.active.set(true);

        if (logger.isDebugEnabled()) {
            if (logger.isTraceEnabled()) {
                logger.trace("Refreshing " + this);
            }
            else {
                logger.debug("Refreshing " + getDisplayName());
            }
        }
        //自定义实现，可以用实际实例替换根属性来源
        initPropertySources();
        // 对属性进行必要的验证
        getEnvironment().validateRequiredProperties();
        if (this.earlyApplicationListeners == null) {
            this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
        }
        else {
            this.applicationListeners.clear();
            this.applicationListeners.addAll(this.earlyApplicationListeners);
        }
        this.earlyApplicationEvents = new LinkedHashSet<>();
    }
```

所以该方法主要是做一些准备工作：设置context启动时间，设置context状态、初始化context environmetn中的占位符、对属性进行必要的校验等。

### obtainFreshBeanFactory()

该方法主要是 创建和初始化 BeanFactory。
