---
title: Dubbo原理之概述
date: 2020-03-21 00:39:50
tags: Dubbo
categories: Dubbo
---

Dubbo 是一款高性能的Java RPC框架。掌握Dubbo也是每个Java攻城狮必备的技能~这篇文章对Dubbo的一些问题进行概述，接下来再由面到点进行分析。

### 为什么要用Dubbo

随着服务化的进一步发展，服务越来越多，服务之间的调用和依赖关系也越来越复杂，诞生了面向服务的架构体系，也因此诞生了一系列的相应技术，如对服务提供、服务调用、连接处理、通信协议、序列化方式、服务发现、服务路由等行为进行封装的框架，就这样为分布式系统的服务治理框架就出现了，Dubbo也就这样产生。

### Dubbo是什么

Dubbo 是一款高性能轻量级的开源RPC框架，提供服务自动注册、自动发现等高效服务治理方案，可以跟Spring框架无缝集成。

### Dubbo的使用场景是什么

1. 透明化的远程方法调用：就像调用本地方法一样调用远程方法，只需要简单的配置，没有任何API侵入
2. 软负载均衡及容错机制：可在内网替代F5等硬件负载均衡器 ，降低成本，减少单点
3. 服务自动注册和发现：不需要写死服务提供方地址，注册中心基于接口名查询服务 提供者的IP地址，并且能够平滑添加和删除服务提供者

### Dubbo的核心功能

1. Remoting：网络通信框架，提供多种对NIO框架抽象封装，包括“同步转异步” 和 “请求-响应”模式的信息交换方式。
2. cluster : 服务框架，提供基于接口方法的透明远程过程调用，包括多协议支持以及软负载均衡、失败容错、地址路由、动态配置等集群支持等。
3. Registry ：服务注册，基于注册中心目录服务，使服务消费方能动态查找服务提供方，使抵制透明，使服务提供方可以平滑增加和减少机器。

### Dubbo的核心组件是什么

1. Provider : 暴露服务的服务提供方
2. Consumer : 调用远程服务消费方
3. Registry : 服务注册与 发现注册中心
4. Monitor : 监控中心与访问调用统计
5. Container :服务运行容器

![](dubbo/architecture.png)

### Dubbo服务注册与发现的流程

服务容器Container负责启动、加载、运行服务提供者

服务提供者Provider在启动时，向注册中心**注册**自己提供的服务

服务消费者Consumer在启动时，向注册中心**订阅**自己所需的服务

注册中心Register返回服务提供者地址给消费者，如果有变更，注册中心基于长连接推送变更数据给消费者。

服务消费者Consumer，从提供者地址列表，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另外一台。

服务消费者Consumer和提供者Provider，在内存中累计调用次数和调用时间，定时每分钟发送一次统计数据给监控中心Monitor。

### Dubbo整体设计有哪些分层

1. **Service 接口服务层** ：该层与业务逻辑相关，根据provider 和 consumer的业务设计对应的接口和实现
2. **Config 配置层**  : 对外配置接口，以ServiceConfig 和 ReferenceConfig 为中心
3. **Proxy 服务代理层** ：服务接口透明代理，生成服务的客户端Stub 和 服务端的 Skeleton，以ServiceProxy为中心，扩展接口为ProxyFactory。
4. **Register 服务注册层** ：封装服务地址的注册和发现，以服务URL为中心，扩展接口为RegisterFactory、Register、RegistryService。
5. **Cluster 路由层** ： 封装多个提供者的路由和负载均衡，并桥接注册中心，以Invoker为中心，扩张个借口为Cluster、Directory、Router、和LoadBlance。
6. **Monitor 监控层** : RPC调用次数和调用时间监控，以Statistic为中心，扩展接口为MonitorFactory Monitor 和 MonitorService。
7. **Protocal 远程调用层** ：封装RPC调用，以Invocation 和 Result 为中心，扩展接口为Protocal Invoker 和 Exporter。
8. **Exchange 信息交换层** ：封装请求响应模式，同步转异步。以Request 和 Response为中心，扩展接口为Exchanger、ExchangeChannel 、ExchangeClient 和 ExchangeServer。
9. **Transport 网络传输层** ：抽象mina 和 netty 为统一接口，以Message为中心，扩展接口为Channel 、Transporter 、Client 、Server 和 Codec
10. **Serialize 数据序列化层** ： 可复用的一些工具，扩展接口为Serialization 、ObjectInput 、ObjectOutput和ThreadPool

![](dubbo/dubbo-framework.jpg)

### Dubbo Monitor的实现原理？

Consumer端在发起调用之前会先走filter链，provider端在接收到请求时，也是先走filter链，然后才进行真正的业务逻辑处理。默认情况下，在consumer和provider的filter链中都会有Monitorfilter。

1. MonitorFilter向DubboMonitor发送数据
2. DubboMonitor将数据进行聚合后，暂存到ConcurrENTMap<Statistics，AtomicReference> statisticsMap，然后使用一个含有3个线程（线程名字 DubboMonitorSendTimer）的线程池每隔1min钟，调用SimpleMonitorService遍历发送statisticsMap中的统计数据，每发送完毕一个，就重置当前的Statistics的AtomicReference。
3. SimpleMonitorService将这些聚合数据塞入BlockignQueue queue中
4. SimpleMonitorService使用一个后台程序（线程名为DubboMonitorAsyncWriteLogThread）将queue中的数据写入文件。
5. SimpleMonitorService还会使用一个含有1个线程的线程池（线程名字 DubboMonitorTimer）每隔5 min钟，将文件中的统计数据画成表。

### Dubbo 有哪些注册中心？

1. Multicast注册中心：不需要任何中心节点，只要广播地址，就能记性服务注册和发现，基于网络中组播传输实现
2. ZK注册中心：基于分布式协调系统ZK实现，采用ZK的watch机制实现数据变更
3. Redis注册中心：基于Redis实现，采用k/v存储，k存储服务名和类型，map中key存储服务url，value为服务过期时间。基于Redis的发布、订阅模式通知数据变更
4. Simple注册中心

#### Dubbo的注册中心集群挂掉，发布者和订阅者之前还能通信吗？

可以通讯。启动Dubbo时，消费者会从ZK拉取注册的生产者的地址接口等数据，缓存在本地。每次调用时，按照本地存储的地址进行调用。

#### Dubbo提供了那些负载均衡策略？

Random LoadBalance ： 随机选取提供者策略，有利于动态调整提供者权重。截面碰撞率高，调用次数越多，分布越均匀。

RoundRobin LoadBalance : 轮询选取提供者策略，平均分布，但是存在请求积累的问题。

LeastActive LoadBalance : 最少活跃调用策略，解决慢满提供者接收更少的请求

ConstantHash LoadBalance : 一致性Hash策略，使相同参数请求总是放到同一提供者，一台机器宕机，可以基于虚拟节点，分摊至其他提供者，避免引起提供者的剧烈变动。

**默认 Random随机调用。**

#### Dubbo集群容错方案有哪些？

Failover Cluster : 失败自动切换，当出现失败，重试其他服务器。通常用于读操作，但重试会带来更长延迟。

Failfast Cluster : 快速失败，只发起一次调用，失败立即报错。通常用于非幂等性的写操作，比如新增记录。

FailSafe Culster : 失败安全，出现异常时，直接忽略。通常用于写入审计日志等操作

FailBack Cluster : 失败自动恢复，后台记录失败请求，定时重发。通常用于消息通知操作。

Forking Cluster ： 并行调用多个服务器，只要有一个成功即返回。通常用于实时性要求较高的读操作，但需要浪费更多的服务资源，可以通过forks = 2 来设置最大并行数。

Broadcast Cluster : 广播调用所有提供者，逐个调用，任意一台报错则报错。通常用于通知所有提供者更新缓存或者日志等本地资源信息。

**默认容错方案是 FailOver Cluster.**

#### Dubbo配置文件是如何加载到Spring中的

Spring容器启动时，会读取到Spring默认的一些schema 以及Dubbo  定义的 shema，每个schema 都会对应一个自己的NamespaceHandler，NamespaceHandler里面通过BeanDefinitionParser来解析配置信息并转化为需要加载的bean对象。

#### Dubbo核心配置有哪些？

| 标签               | 用途         | 解释                                                         |
| :----------------- | :----------- | :----------------------------------------------------------- |
| dubbo:service/     | 服务品质     | 用于暴露一个服务，定义服务的元信息，一个服务可以用多个协议暴露，一个服务也可以注册到多个注册中心 |
| dubbo:reference/   | 引用配置     | 用于创建一个远程服务代理，一个引用可以指向多个注册中心       |
| dubbo:protocol/    | 协议配置     | 用于配置提供服务的协议信息，协议由提供方指定，消费方被动接受 |
| dubbo:application/ | 应用配置     | 用于配置当前应用信息，不管该应用是提供者还是消费者           |
| dubbo:module/      | 模块配置     | 用于配置当前模块信息，可选                                   |
| dubbo:registry/    | 注册中心配置 | 用于配置连接注册中心相关信息                                 |
| dubbo:monitor/     | 监控中心配置 | 用于配置连接监控中心相关信息，可选                           |
| dubbo:provider/    | 提供方配置   | 当 ProtocolConfig 和 ServiceConfig 某属性没有配置时，采用此缺省值，可选 |
| dubbo:consumer/    | 消费方配置   | 当 ReferenceConfig 某属性没有配置时，采用此缺省值，可选      |
| dubbo:method/      | 方法配置     | 用于 ServiceConfig 和 ReferenceConfig 指定方法级的配置信息   |
| dubbo:argument/    | 参数配置     | 用于指定方法参数配置                                         |

#### Dubbo超时设置有哪些方式？

Dubbo超时设置有两种方式：

1. 服务提供者端设置超时时间，在Dubbo的用户文档中，推荐如果能在服务端多配置就尽量多配置，因为服务提供者比消费者更清楚自己提供的服务特性
2. 服务消费者端设置超时时间，如果在消费者端设置了超时时间，以消费者端为主，即优先级更高。因为服务调用方设置超时时间控制性更灵活，如果消费方超时，服务端线程不会定制，会产生警告。

#### 服务调用超时会怎么样？

Dubbo在调用服务不成功时，默认会重试两次。

#### Dubbo使用什么通信框架？

默认使用Netty作为通讯框架。

#### Dubbo支持哪些协议，他们的优缺点有哪些？

1. Dubbo :  单一长连接和NIO异步通讯，适合大并发小数据量的服务调用，以及消费者远大于提供者。传输协议TCP，异步Hessian序列化。Dubbo推荐使用dubbo协议。
2. RMI : 采用JDK标准的RMI协议实现，传输参数和返回参数对象需要实现Serializable接口，使用Java标准序列化机制，使用阻塞式短连接，传输数据包大小混合，消费者和提供者个数差不多，可传文件，传输协议TCP。
3. WebService : 基于WebService的远程调用协议，继承CXF实现，提供和原生WebService的互操作。多个短连接，基于HTTP传输，同步传输，适用系统集成和跨语言调用
4. HTTP : 基于Http表单提交的远程调用协议，适用Spring 的HttpIncoke实现。多个短连接。
5. Hessian : 继承Hessian服务，基于Http通讯
6. Memcache ： 基于 Memcache实现的 RPC 协议。
7. Redis ：基于 Redis 实现的RPC协议。

#### Dubbo支持服务降级吗？

通过dubbo：reference 中设置mock = "return null"。 mock的值也可以修改为true，然后再跟接口同一个路径下实现一个Mock类，命名规则是”接口名称+Mock“，然后在Mock类中实现自己的降级逻辑。

#### Dubbo如何优雅停机？

Dubbo是通过JDK的ShutDownHook来王城优雅停机。

#### Dubbo SPI 和 Java SPI区别？

JDK SPI ：

JDK表中的SPI会一次性加载所有的扩展实现，如果有的扩展很耗时，但也没用上，会很浪费资源。所有只希望加载某个实现，不现实

Dubbo SPI:

1.对Dubbo 进行扩展，不需要改变Dubbo源码

2.延迟加载，可以一次加载自己想要加载的扩展实现

3.增加了堆扩展点IOC 和 AOP的支持，一个扩展点可以直接setter注入其扩展点

4.Dubbo 的扩展机制能很好的支持第三方IOC容器，默认支持Spring bean。

#### Dubbo支持哪些序列化方式？

Dubbo默认使用Hessian序列化，还有Dubbo 、FastJson 、Java自带序列化

#### Dubbo在安全方面有哪些措施？

Dubbo 通过Token令牌防止用户绕过注册中心直连，然后在注册汇中心上管理授权

Dubbo还提供黑白名单，来控制服务所允许的调用方。

#### 服务的调用是阻塞的吗？

默认是阻塞的，可以异步调用，没有返回值的可以这么做。Dubbo基于NIO的非阻塞实现并行调用，客户端不需要启动多线程即可完成并行调用多个远程服务，相对多线程开销较小，异步调用会返回一个Future对象。

#### 服务提供者能实现失效踢出的原理是什么？

服务失效踢出基于ZK的临时节点原理

#### Dubbo使用过程中都遇到了那些问题？

注册中心找不到对应的服务，检查service实现类是否添加了@Service注解。无法连接到注册中心，检查配置文件对应的ip是否正确。

#### 为什么要有RPC?

HTTP接口在接口不多，系统与系统之间交互较少的情况下，解决信息孤岛初期常使用的一种通信手段。优点就是简单 直接 开发方便。利用现成的http协议进行传输。但是一个大型网站，内部子系统较多，接口非常多的情况下，RPC框架的好处就显现出来，首先是长连接，不必每次通信都要像http一样去三次握手，减少了网络开销。其次RPC框架一般都有注册中心，有丰富的监控管理；发布、下线接口、动态扩展等，对调用方是无感知的、统一化的操作。第三个来说就是安全性。

socket只是一个简单的网络通信方式，只是创建通信双方的通信通道，而要实现RPC功能，还要对其进行封装、以实现更多功能。RPC一般配合netty框架，netty内部封装了socket，spring自定义注解来编写轻量级框架。

#### 什么是 RPC?

RPC 远程过程调用协议，它是一种 网络从远程计算机程序上请求服务，而不需要了解底层网络技术的协议。简言之，RPC使得程序能够像调用本地服务一样，去访问远端系统资源。





> 参考列表：
>
> 1. Dubbo官网
> 2. https://developer.aliyun.com/ask/274788?utm_content=g_1000108236