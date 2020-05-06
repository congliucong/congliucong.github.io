---
title: Redis知识归纳(二)过期策略
date: 2020-04-02 22:03:50
tags: Redis
categories: Redis
---

### Redis的数据结构

1. string

   最简单的类型，可以做为简单的kv缓存

2. list

有序队列，可以存储一些列表型的数据结构、通过lrange实现分页、也可以作为队列

3. hash

类似于map的接口、一般用来存储结构化的对象

4. set

无序结合，自动去重，可以用来计算交集、并集、差集

5. sort set

排序的set，通过sorce自动排序。数据结构是跳跃表

### redis过期策略

redis 的过期策略是 ：**定期删除 + 惰性删除**。

定期删除指的是：redis每秒10次，即每100ms**随机抽取**一些设置了过期时间的key，检查其是否过期，如果过期，就从redis中删除。如果有多于25%的keys过期，那么就再来一次。

惰性删除指的是：当获取 某个key的时候，如果key已经过期，那么不返回任何东西，并且删除。

当然以上策略是不能够保证所有过期的key都删除的，因此还需要内存淘汰机制：

1. volatile-lru（Least recently used） : 从设置了过期时间的集合中删除最近最久未使用的。
2. allkeys-lru : 所有key中删除最久未使用的。
3. volatile- lfu （Least frequently used）: 从设置了过期时间的集合中删除最近最少使用的。
4. allkeys-lfu : 从所有keys中删除最近最少使用的。
5. volatile-random : 从设置了过期时间的结合中随机删除。
6. allkeys-random : 从所有keys中随机删除
7. volatile-ttl : 设置了过期时间里面ttl最少的。
8. 啥也不做

