---
title: Java基础之AQS
date: 2020-04-26 21:39:24
tags: Java
categories: Java基础
---

之前太多次提到AQS了，JDK1.7的ConcurrentHashMap的segment继承ReentrantLock，而ReentrantLock是基于AQS实现的。线程池的Worker类继承线程池，实现了不可重复的锁机制，从而通过能否获取锁判断线程是否处于运行状态。可见AQS的重要性。

而且，整个JUC包都是基于AQS实现的，而且AQS是面试必问的，咋说都要学懂学透。之前也看过好多次，这次以文字记录，加深印象。![](AQS/006FAE26.png)

<!-- more -->

### AQS简介

JDK1.5之后，大神**Doug Lea**编写了JUC包，从而一定程度上代替了Synchroniezed。而**AQS（AbstractQueuedSynchronizer）**则是JUC包下的核心组件，JUC包下大部分同步组件都是基于AQS的。AQS的设计模式采用了**模板设计模式**，主要使用方式是继承，子类通过继承AQS并实现它的核心抽象方法，即可轻松的管理同步问题。

从功能上来讲，AQS主要分为两种：**独占**和**共享**。JUC下的ReentrantLock就是一种独占锁，当一个线程获取锁之后，其他线程只能阻塞等待持有锁的线程释放锁。ReentrantReadWriteLock 中的读锁，则是一种共享锁，多个线程可以共享一把锁，同时执行任务。

### AQS原理

AQS的核心思想是：如果请求的共享资源空闲，则当前请求线程将自己设置为有效的工作线，同时锁定共享资源，如果其他线程获取同步状态失败，则将其他线程加入到某个容器中并阻塞，等同步状态释放时，则把阻塞的线程唤醒，使其可以再次尝试获取锁。

而这个AQS是通过CLH队列的变体实现线程排队阻塞的，将暂时获取不到的线程加入到队列中。

CLH是Craig、Landin and Hagersten队列，是一个单向链表。而AQS中的队列是**CLH变体的虚拟双向队列**。队列中的每一个元素就是一个Node节点。

![](AQS/7132e4cef44c26f62835b197b239147b18062.png)



```java
static final class Node {
    //共享节点
    static final Node SHARED = new Node();
    //独占节点
    static final Node EXCLUSIVE = null;
    //超时或者中断，则会被设置为取消
    static final int CANCELLED =  1;
    //后继节点处于等待状态
    static final int SIGNAL    = -1;
    //这个是Condition使用的。当其他线程对Condition调用了singal之后，会把节点从等待队列转移到同步队列
    static final int CONDITION = -2;
 	//表示releaseShared需要被传播给后续节点（仅在共享模式下使用）
    static final int PROPAGATE = -3;
 	//等待状态
    volatile int waitStatus;
 	//前驱节点
    volatile Node prev;
 	//后驱节点
    volatile Node next;
 	//获取同步状态的线程
    volatile Thread thread;
 	//存储condition队列中的后继节点
    Node nextWaiter;
 	
    final boolean isShared() {
        return nextWaiter == SHARED;
    }
 	//
    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }

    Node() {    // Used to establish initial head or SHARED marker
    }

    Node(Thread thread, Node mode) {     // Used by addWaiter
        this.nextWaiter = mode;
        this.thread = thread;
    }

    Node(Thread thread, int waitStatus) { // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
```



```java
/**
 * The synchronization state.
 */
private volatile int state;
```

AQS内部通过维护一个volatile修饰的state变量。对于不同的子类，有不同的意义。

### ReentrantLock

我们从ReentrantLock开始，以ReentrantLock原理来逐步解剖AQS。

S

首先，我们明确ReentrantLock是一种独占锁。

其次，ReentrantLock的几个内部类中：

1. sync继承AQS。
2. FairSync则继承Sync，是一种**公平锁**
3. NonfairSync继承Sync，是一种**非公平锁**。

![](AQS/image-20200426232017556.png)

#### ReentrantLock获取锁

公平锁：

![](AQS/image-20200426233506614.png)

非公平锁：

![](AQS/image-20200426233532377.png)

可以看出，公平锁和非公平锁获取锁的时候，区别在于，非公平模式会尝试用CAS将state修改为1，并讲当前线程设置为独占线程，失败后才会执行acquire(1)方法。而公平模式则会直接调用acquire(1)方法。

##### AQS定义acquire方法

acquire方法是AQS里面的方法，

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

这方法主要执行：

1. 尝试获取独占锁tryAcquire(arg)，这个方法需要同步组件自己实现。

2. 获取成功则返回
3. 不成功，则先执行addWaiter将当前线程加入队列
4. 然后acquireQueued自旋获取锁，并且返回当前下次线程在等待过程中是否中断过
5. selfInterrupt()产生中断。

我们接着对获取锁整个过程进行分析。

##### 子类定义tryAcquire

公平锁：

```java
    /**
     * Fair version of tryAcquire.  Don't grant access unless
     * recursive call or no waiters or is first.
     */
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        //获取当前state
        int c = getState();
        //如果state等于0，说明没有加锁
        if (c == 0) {
            //判断有没有前驱节点，如果没有则尝试CAS修改state,获取锁，并设置当前线程所有
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        //如果state不等于0，说明已经有人获取锁了，则判断获取锁的是不是自己
        else if (current == getExclusiveOwnerThread()) {
            //如果是自己，则重入次数+1，
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            //设置state，此时state表示获取锁的次数
            setState(nextc);
            return true;
        }
        return false;
    }
}
```



非公平锁：

```java
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}

final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

对比上面两个方法，发现公平锁和非公平锁方法只有一个区别，公平锁需要多加***!hasQueuedPredecessors()***，我们过会在来分析这个方法，先分析当首次尝试获取锁不成功，是如何入队和自旋获取锁的。







































































> 参考资料
>
> 1. http://cmsblogs.com/?cat=151&paged=2
> 2. https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html

