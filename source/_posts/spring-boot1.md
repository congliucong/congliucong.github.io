---
title: spring boot源码分析（一）：启动过程
date: 2020-05-18 15:20:19
tags: SpringBoot
categories: Spring
---

从这篇文章开始，我们开始从源码角度分析SpringBoot的一些相关原理。

<!-- more -->

### SpringBoot的入口

```java
// Spring Boot 应用的标识
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        // 程序启动入口
        // 启动嵌入式的 Tomcat 并初始化 Spring 环境及其各 Spring 组件
        SpringApplication.run(Application.class,args);
    }
}
```

#### SpringApplication的构造方法

```java
    /**
     * Static helper that can be used to run a {@link SpringApplication} from the
     * specified sources using default settings and user supplied arguments.
     * @param primarySources the primary sources to load
     * @param args the application arguments (usually passed from a Java main method)
     * @return the running {@link ApplicationContext}
     */
    public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
        return new SpringApplication(primarySources).run(args);
    }
```

当程序开始执行run方法时，会先调用SpringApplication的构造方法。该构造方法会创建一个新的SpringApplication实例。

```java
    public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
        // 初始化资源加载器，默认为null
        this.resourceLoader = resourceLoader;
        // 断言主要加载资源类不能为null，否则报错
        Assert.notNull(primarySources, "PrimarySources must not be null");
        // 初始化主要加载资源类集合并去重
        this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
        // 设置当前WEB应用类型，主要有是三种： NONE,SERVLET,REACTIVE
        this.webApplicationType = WebApplicationType.deduceFromClasspath();
        // 设置应用上下文初始化器，主要从"META-INF/spring.factories"中
        // 读取ApplicationContextInitializer类的实例名称集合并去重，所以一共7个
        setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
        // 设置监听器，主要从"META-INF/spring.factories"中
        // 读取ApplicationListener类的实例名称集合并去重，所以一共11个
        setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
        // 找到Main方法所在类，并赋值给mainApplicationClass
        this.mainApplicationClass = deduceMainApplicationClass();
    }
```

#### Run方法

SpringBoot启动的运行方法，可以看出主要是各种运行环境的准备工作。

```java
    public ConfigurableApplicationContext run(String... args) {
        // 1. 创建并启动计时监控类
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();
        // 2. 初始化应用上下文和异常报告集合
        ConfigurableApplicationContext context = null;
        Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
        // 3. 设置系统属性“java.awt.headless”的值，默认为true.用于运行headless服务器，也就是没有显示屏、鼠标的系统，很多监控软件如jconsole需要将该值设为true。
        configureHeadlessProperty();
        // 4. 创建所有spring运行监听器并发布应用启动事件
        // 简单的说就是获取SpringApplicationRunListener类型的实例（EventPublishingEunListener对象）
        SpringApplicationRunListeners listeners = getRunListeners(args);
        listeners.starting();
        try {
            // 5. 初始化默认应用参数类
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
            // 6. 根据运行监听器和应用参数来准备spring环境
            ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
            // 将要忽略的bean参数打开
            configureIgnoreBeanInfo(environment);
            // 7. 创建banner打印类
            Banner printedBanner = printBanner(environment);
            // 8. 创建引用上下文，可以理解为创建一个容器
            context = createApplicationContext();
            // 9. 准备异常报告器，用来支持报告关于启动的错误
            exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
                    new Class[] { ConfigurableApplicationContext.class }, context);
            // 10 准备应用上下文，该步骤包括一个非常关键的操作，
            // 将启动类注入到容器，为后续开启自动化提供基础
            prepareContext(context, environment, listeners, applicationArguments, printedBanner);
            // 11。 刷新应用上下文
            refreshContext(context);
            // 12。应用上下文刷新后置处理，做一些扩展功能
            afterRefresh(context, applicationArguments);
            // 13。停止计时监控类
            stopWatch.stop();
            // 14. 输出日志记录执行主类名、时间信息
            if (this.logStartupInfo) {
                new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
            }
            // 15. 发布应用上下文启动监听事件
            listeners.started(context);
            // 16. 执行所有Runner运行期
            callRunners(context, applicationArguments);
        }
        catch (Throwable ex) {
            handleRunFailure(context, ex, exceptionReporters, listeners);
            throw new IllegalStateException(ex);
        }

        try {
            // 17. 发布应用上下文就绪事件
            listeners.running(context);
        }
        catch (Throwable ex) {
            handleRunFailure(context, ex, exceptionReporters, null);
            throw new IllegalStateException(ex);
        }
        //18. 返回应用上下文
        return context;
    }
```
