---
title: Java基础之JUC(三)ReentrantReadWriteLock
date: 2020-04-28 23:26:05
tags: Java
categories: Java基础
---

前面介绍了AQS的共享模式和独占模式，以及Condition功能，共享锁以CountDownLatch为例，排他锁则是以ReentrantLock为例。这篇文章就一起学习ReentrantReadWriteLock，在ReentrantReadWriteLock中包括读锁和写锁，读锁可以多线程访问，属于共享锁，写锁属于排它锁，更改的时候不允许其他线程访问。同时写锁也提供了Condition机制。不允许读写共存，但是可以读读共存。

<!-- more -->

### ReentrantReadWriteLock原理

很多场景下，大部分时间提供读服务，而写服务占用的时间比较少。因为如果读操作相关互斥的话，那么势必会带来性能的下降。因此读写锁适合于读多写少的场景 ，能够提高并发效率。

ReentrantReadWriteLock实现了ReadWriteLock接口，该接口维护着两把锁，一把读锁，一把写锁。

```java
public interface ReadWriteLock {
    Lock readLock();
    Lock writeLock();
}
```

因此，ReentrantReadWriteLock通过readLock()/writeLock()方法对外提供读锁/写锁。同时内部类Sync继承于AQS，读锁、写锁都是依靠Sync来实现的。

![](AQS4/image-20200429000753077.png)

同时，我们还记得介绍AQS提到，用volatile修饰的state，不同子类实现有不同的含义。在ReentrantReadWriteLock中，就是通过state来维护读锁、写锁的状态。采用“按位切割使用”的方式来维护这个变量。

因为state是int型，共有32bit，ReentrantReadWriteLock将其分为两部分，高16表示读，低16位表示写。

![](AQS4/image-20200429001929062.png)

可以看出来，通过位运算来 计算读写锁的装填。用state & 0x0000FFFF（抹去高16位）表示写状态，state >>> 16 (右移 16位，抹去低16位)表示读状态。 