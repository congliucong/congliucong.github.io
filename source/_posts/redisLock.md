---
title: 分布式之分布式锁
date: 2020-04-19 21:21:17
tags: 分布式锁
categories: 分布式
---

这篇文章，主要的分布式锁相关知识体系进行总结。因为啥呢，Redis不仅是最常用的中间件，也是面试高频话题。谈到Redis，绕不开的必然是分布式锁。这篇文章主要关注点由单点Redis实现分布式锁，延伸到集群Redis实现分布式锁，再到Zookeeper实现分布式锁，以及对Redisson的部分源码解析。还是那句话，把知识由点总结成面，加深理解，让各个知识点不再是一个个孤岛。

<!-- more -->

### 单点Redis分布式锁

在单点部署的Redis中，如何使用Redis来作为分布式锁呢？关键点主要有三：

* **原子命令加锁**

  > SET key random_value NX PX 30000

  NX:表示只有当要设置的 key值不存在时，才能Set成功。这样可以保证只有第一个用户能够成功设置锁。

  PX 30000: 表示该锁存活时间为30秒，即使没有主动释放锁，该锁在30秒后也会自动释放。

* **设置value，要设置随机数**

为什么设置的value要是随机数？

这是因为 释放锁的时候要检查key是否存在，如果存在还要进行value的比较，看这个value值是不是自己设的，一样才能释放锁，这样可以避免 A设置的锁却被B释放的情况。

* **释放锁要使用lua脚本**

释放锁整个过程要 获取--->判断--->删除 三个步骤，这三个步骤不是原子性的，所以为了保障原子性，要是用lua脚本。



但是，单点部署的Redis节点，当实例挂掉之后，Redis锁会全部失效，导致系统不可用。这个时候就需要Redis的集群方式部署实现Redis的高可用。

### Redis集群

Redis实现集群有多种方案。例如 主从模式 哨兵模式 cluster模式 利用中间件代理等。这里只大概描述主要的集群方式，详细内容以后再用专门文章来描述Redis如何实现高可用。

* 主从模式

主从模式包括一个master和多个salve，客户端对master进行读写操作，对slave进行读操作。master写入的数据会自动同步给salve，但是主从模式最大的问题是 当主节点挂掉之后，需要手动切换salve为master，因此实际生产中很少用到。

* 哨兵模式

哨兵模式基于主从复制模式，多了哨兵来监控和自动进行故障迁移。该模式下，哨兵会监控主节点，当主节点异常时，会标记为主观下线或客观下线，并进行主从切换。因此可以保障高可用。

* cluster集群

这是Redis官方推出的集群方案。保证了高并发，整个集群通过不同的槽位分担所有数据，不同的key会分发到不同的Redis。



但是不管是什么集群方案，都会存在一种极端情况下的问题：当master节点上新加了锁，但是由于主从异步通信同步，数据还没有同步到slave上，此时master挂掉，master上的锁就会失效。等新的master重新设好之后，可以再次设置同样的锁，这样就会出现同一把锁设置两次的问题。一把锁，不管在什么情况下，只能保证设置一次，不然锁就失去了锁的意义。

### Redlock算法

为了解决集群下可能出现的问题，官方提出了RedLock算法。现在假设有5台Redis master节点。
> 以下获取锁步骤来自http://redis.cn/topics/distlock.html

* 获取当前Unix时间，以毫秒为单位。
* 依次尝试从N个实例，使用相同的key和随机值获取锁。在步骤2，当向Redis设置锁时,客户端应该设置一个网络连接和响应超时时间，这个超时时间应该小于锁的失效时间。例如你的锁自动失效时间为10秒，则超时时间应该在5-50毫秒之间。这样可以避免服务器端Redis已经挂掉的情况下，客户端还在死死地等待响应结果。如果服务器端没有在规定时间内响应，客户端应该尽快尝试另外一个Redis实例。
* 客户端使用当前时间减去开始获取锁时间（步骤1记录的时间）就得到获取锁使用的时间。当且仅当从大多数（这里是3个节点）的Redis节点都取到锁，并且使用的时间小于锁失效时间时，锁才算获取成功。
* 如果取到了锁，key的真正有效时间等于有效时间减去获取锁所使用的时间（步骤3计算的结果）。
如果因为某些原因，获取锁失败（没有在至少N/2+1个Redis实例取到锁或者取锁时间已经超过了有效时间），客户端应该在所有的Redis实例上进行解锁（即便某些Redis实例根本就没有加锁成功）

上面步骤意思就是，只要在大多数节点上加锁成功，则获取锁成功。但是这样也并不能解决故障重启后带来的锁安全问题。假如一共有5台实例且Redis没有使用持久化策略，一个客户端获取到3个实例的锁。此时其中一个客户端实例重启，在这个时间点，可能出现3个节点没有设置锁。此时如果有另外一个客户端来设置锁，锁有可能被再次获取，这样就违背了锁的互斥原则。

即使启用了AOF持久化策略，AOF默认情况下每秒写一次磁盘，即fsync操作，因此最坏情况下丢失一秒的数据。但是，即使只有一秒，也有可能出现未来得及持久化，仍会出现重复加锁的情况。

因此Redis的作者提出延时重启的方案，即节点崩溃之后，不要立即重启它，而是等待超过锁的过期时间后重启，保证自身上的锁都过期。但是这样做，有可能会让系统彻底转化为不可用状态。例如大部分redis实例崩溃，系统在这个时间段内任何锁都无法加锁成功。

因此，可以看出，在集群情况下，没有一种完美的方案，任何方案都会有极端的情况出现。如果系统可以接受极端情况下可能发生的问题，这种方案也是可以使用的，这就要看系统的需求取舍。

### Redisson实现原理

先来看一下网上找的一张Redisson原理图，我觉得画的还是挺清晰的
![](redisLock/redisson.png)

#### 代码示例
![](redisLock/code.jpg)
相比看到这里，肯定会有疑问，基于我们上面说的
> SET key random_value NX PX 30000

为什么RLock lock = client.getLock("lcc");lock.lock();就能加锁成功呢？value又是多少呢？ 设置的过期时间又是多少？
带着这些疑问，我们进入源码一探究竟。
首先lock会执行到这里:
> org.redisson.RedissonLock#lock(long, java.util.concurrent.TimeUnit, boolean)
![](redisLock/lock1.jpg)

这里可以看到，将当前线程的线程ID作为参数传入，后续会用到这个ID。
接着继续Debug，会进入：
![](redisLock/lock2.jpg)
> org.redisson.RedissonLock#tryAcquireAsync
> 其中一个参数为：commandExecutor.getConnectionManager().getCfg().getLockWatchdogTimeout()，可以看到这个时间默认为private long lockWatchdogTimeout = 30 * 1000;

这关键参数就是 设置锁的默认过期时间。
继续往下看
![](redisLock/lock3.jpg)

其中 internalLockLeaseTime 是过期时间，而getLockName(threadId)是拼接ID+线程ID
![](redisLock/lock5.jpg)

* 而ID是从commandExecutor.getConnectionManager()中获取，
![](redisLock/lock6.jpg)

* 继续网上找，connectionManager是在Redisson构造函数时创建
![](redisLock/lock7.jpg)
* 一路跋山涉水终于找到 ID的来源是 UUID。
![](redisLock/lock8.jpg)

由此可以推断，虽然获取锁时没有显示执行value，但是Redisson会默认帮我们生成UUID + 线程ID 作为 value,过期时间为默认30s。我们等会进行验证。

我们接着分析加锁这段的这段lua脚本。
```lua
if (redis.call('exists', KEYS[1]) == 0) then  
  redis.call('hincrby', KEYS[1], ARGV[2], 1);  
  redis.call('pexpire', KEYS[1], ARGV[1]);  
  return nil;  
end; 
if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then 
  redis.call('hincrby', KEYS[1], ARGV[2], 1); 
  redis.call('pexpire', KEYS[1], ARGV[1]); 
  return nil; 
end; 
return redis.call('pttl', KEYS[1]);
```
![](redisLock/lock4.jpg)
可以看到，这段脚本中，keys：是我们getLock时设置的key，params是lua脚本的参数，其中ARGV[1]是 30000，ARGV[2]是 UUID+线程ID。

第一段脚本中 判断 key是否存在，如果不存在，则使用hincrby命令。
![](redisLock/hincrby.jpg)
可以看出来，value的数据结构是hash,hincrby 将哈希表 key 中域 field 的值设置为1。