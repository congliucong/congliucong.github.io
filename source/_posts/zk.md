---
title: Zookeeper原理概述
date: 2020-05-07 13:19:42
tags: 分布式事务
categories: 分布式
---

之前讨论研究Redis分布式锁的时候，提到过Zookeeper分布式锁。这篇文章来研究一下zk的相关原理。

<!-- more -->

## Zookeeper有什么用？

我们能接触ZK最常见的场景就是作为Dubbo的注册中心。这正是因为ZK是一个分布式协调服务，可以用来服务发现、分布式锁、分布式领导选举、配置管理等场景。

### Node节点

这所有功能的基础是因为ZK维护着一个类似文件系统的数据结构。整体上可以看做是一棵树，每个节点称作一个ZNode。有四种类型的znode:
1. PERSISTENT 持久化目录节点
客户端与zk断开连接后，依然存在
2. PERSISTENT_SEQUENTIAL 持久化顺序编号目录节点
客户端与zk断开连接后，依然存在，并且每个节点有顺序编号
3. EPHEMERAL 临时节点
客户端与zk断开后节点就被删除
4. EPHEMERAL_SEQUENTIAL 临时顺序编号目录节点
客户端与zk断开连接后，节点就被删除，只不过这些节点都有顺序

### Watcher机制

zk的另外一个机制就是 通知机制。ZK允许客户端在指定节点上注册一些watcher，当目录节点发生变化时，如数据修改、删除等，zk会将事件通知到感兴趣的客户端上，这也是实现分布式协调服务的重要特性。

Watcher机制主要包括客户端、客户端WatcManager 和 ZooKeeper服务器三部分。在具体工作流程上，简单的说，就是客户端在向zk服务器注册watcher的同时，将watcher对象存储在客户端的watchManager中。当服务器出发watcher事件后，会向客户端发送通知，客户端线程从watchManager中取出对应的Watch对象来执行回调逻辑。

代码实现：

在WatchManager中是使用HashMap来存放Watcher对象。
```java
public class WatchManager {

    private final HashMap<String, HashSet<Watcher>> watchTable =
        new HashMap<String, HashSet<Watcher>>();
    }
```

执行watcher对象的回调函数是执行watcher的process方法。
```java
public interface Watcher {
     abstract public void process(WatchedEvent event);
}
```

### ZooKeeper服务器角色

ZK集群是一个基于主从复制的高可用集群，每个服务器可以承担如下三种角色中的一种：
1. Leader 
一个ZK集群同一时间只会有一个实际工作的Leader,它会发起并维护与各个Follower及Observer间的心跳。所有写操作必须通过Leader完成再由Leader将写操作广播给其他服务器。
2. Follower
一个ZK集群可能有多个Follower，它会响应Leader的心跳。Follower可以处理并返回客户端的读请求，并且将写请求转发给Leader处理，并且负责在Leader处理写请求时对写请求进行投票。
3. Observer
与Follower相同，但是没有投票权。

### ZAB协议

Zookeeper是通过Zab协议来保证分布式事务的最终一致性。
Zab协议主要包括两大模式：
1. 原子广播
2. 崩溃恢复

### Zookeeper读、写操作

#### 写Leader

1.客户端向Leader发起写请求。
2.Leader将写请求以Proposal的形式发给所有Follower并等待ACK。
3.Follower收到Leader的Proposal返回ACK
4.Leader得到过半数的ACK后，向所有的Follower和Observer发送Commit.
5.Leader将处理结果返回给客户端。

#### 写Follower/Observer

Follower和Observer都可以接受写请求，但是不能直接处理，而是需要将写请求转发给Leader处理。其他流程与直接写Leader没有区别。

#### 读操作

Leader/Follower/Observer都可以直接处理读请求，从本地内存中读取数据并返回给客户端即可。因为读请求不需要进行转发，因此Follower/Observer越多，整体可处理的读请求就越大。

### Leader选举

ZK使用基于TCP的FastLeaderElection算法来实现leader选举。在介绍FastLeaderElection前需要先介绍几个概念：

1. myid
zk搭建集群时，需要为每个zk服务器配置全局唯一的myid.
2. zxid
zk状态的每一次改变，都对应这一个递增的Transaction id，该id就是zxid。zk使用一个64位的数来表示，高32位表示Leader的epoch，从1开始，每次选出新的Leader，epoch加1。低32位为该epoch内的序号，每次epoch变化，都将低32位的序号重置。这样保证zkid的全局递增型。
3. 选票数据结构
每个服务器在进行领导选举时，都会发送如下关键信息：
  logicClock:表示该服务器发起的第几轮投票
  state: 当前服务器的状态
  self_id：当前服务器的myid
  seif_zxid : 当前服务器上保存的数据的最大zxid
  vote_id：被推举服务器的myid
  vote_zxid : 被推举服务器上保存的数据的最大zxid  
4. 投票流程
  1.所有有效的投票必须在同一轮次。每个服务器开始新一轮投票钱，先将logicClock进行自增操作。
  2.每个服务器在广播自己的选票前，会将自己的投票箱清空。票箱中只会记录每一个投票者的最后一票。
  3.每个服务器最开始都是通过广播把票投给自己
  4.服务器会尝试从其他服务器获取投票，并计入自己的票箱。如果无法获取，则判断自己是不是短线。如果是断线了，则再次发送自己的投票。
  5.每次收到选票，判断选票的logicBack和自己logicBack，如果外部的logicBack较大，说明该服务器的选举轮次落后于其他服务器的选举轮次，那么清空自己的投票箱并将自己的logicBack更新为收到的logicBack，然后对比自己的之前的选票与收到的投票以确定是否需要变更自己的投票，最终将自己的投票广播出去。如果外部的logicBack小，则忽略该选票。如果相等，则进行选票PK。
  6.选票PK是基于（self_id,self_zxid）与（vote_id，vote_zxid）的对比：
    如果外部的vote_zxid比较大，那么将自己选票的vote_zxid 和 vote_myid更新为收到选票的vote_zxid 和 vote_myid并广播出去。同时将收到的票和自己更新后的票放入自己的票箱。
    如果vote_zxid一样，则比较vote_myid，谁的myid大，则广播谁的。
7.最后统计选票，如果有过半服务器认可自己的选票，那么终止投票，否则继续接收其他服务器选票。
8.最后服务器更新自己的状态，若过半的票投给自己，就将自己的服务器状态更新为Leading，否则将自己的状态更新为Following。




















https://www.ibm.com/developerworks/cn/opensource/os-cn-apache-zookeeper-watcher/index.html

