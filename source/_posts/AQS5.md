---
title: Java基础之JUC(五)Semaphore
date: 2020-04-29 14:05:01
tags: Java
categories: Java基础
---

今天继续学习JUC下的工具类------Semaphore。Semaphore是一个信号计数量，与CountDownLatch一样，也是一个共享锁。不同的是，CountDownLatch是一个或多个线程await，其他线程countdown，当state为0后，await的线程继续执行，即一个线程或多个线程需要等待其他线程执行完之后才能执行。而Semaphore维护着一个许可集，每个线程acquire成功，许可数减1，然后执行程序，执行完毕后release，如果acquire失败，说明许可证没有了，因此阻塞，等待许可重新释放。类似于数据库连接池。

<!-- more -->

### Semaphore原理

Semaphore信号量适用于资源有限的场景。如果资源充足，则可以执行程序，否则等待。以停车场类比，停车场的车位数就是信号量，只有有车位，才能停车，否则只能等待。

在Semaphore同样有一个内部类Sync继承AQS，此时AQS的state值作为许可数量使用。

Semaphore提供两个构造函数:

```java
    public Semaphore(int permits) {
        sync = new NonfairSync(permits);
    }

        public Semaphore(int permits, boolean fair) {
        sync = fair ? new FairSync(permits) : new NonfairSync(permits);
    }
```

可以看出，Semaphore支持两种模式：公平模式、非公平模式。

### Semaphore源码分析

#### acquire()信号量获取

acquire调用AQSacquireSharedInterruptibly方法，然后调用AQS子类Sync的tryAcquireShared方法。

```java
    public void acquire() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }


    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }
```

公平模式：

```java
        protected int tryAcquireShared(int acquires) {
            for (;;) {
                if (hasQueuedPredecessors())
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
    }
```

非公平模式：

```java
        protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires);
        }

        final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
    }
```

可以看出，公平模式与非公平模式区别在于 hasQueuedPredecessors() 判断队列中有没有在等待的节点。而在ReentrantLock公平模式下获取锁，也会进行这个判断。

回到这个tryAcquireShared方法里，首先获取state值，这个值的大小代表剩下许可的数量。减去申请的数量后，如果剩余的许可小于0，则retrun remaining。如果剩下的许可大于0，那么将state值重新设置，成功则返回remaining。

```java
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
```

当tryAcquireShared(arg)返回值大于0，那么程序继续执行。如果返回值小于0，说明没有许可了，则进入doAcquireSharedInterruptibly(arg)方法，入队阻塞。

```java
    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }
```

执行到这个方法，说明state小于等于0，没有许可证。这个方法跟CountDownLatch之后await后，发现state不为0后一样。
```java
    private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        //创建SHARED节点，并入队（头结点为虚节点，无实际意义）
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) {
                //如果当前节点有前驱节点
                final Node p = node.predecessor();
                //如果前驱节点为头节点
                if (p == head) {
                    //则尝试抢锁
                    int r = tryAcquireShared(arg);
                    //如果r > 0，说明有许可可用
                    if (r >= 0) {
                        //将自己设置为头结点，并往后传播
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                //如果前驱节点不是头结点，则判断是否中断
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            //如果发生异常，则取消节点
            if (failed)
                cancelAcquire(node);
        }
    }
```

#### release()信号量释放

Semaphore调用release，release调用AQS的 releaseShared，releaseShared再调Sync的tryReleaseShared。

```java
    public void release() {
        sync.releaseShared(1);
    }

    //调用AQSreleaseShared
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```

这里释放就是将许可归还，因此是state+释放的量。
```java
        protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }
```

如果释放成功，则唤醒队列中的节点。这个方法多次出现，目前已经在CountDownLatch的countdown、ReentrantReadWriteLock的ReadLock的释放出现过，以及上述首次抢锁失败后进入队列后如果前面节点是头结点再次抢锁成功后的setHeadAndPropagate方法里面。这里就不在赘述了。因此，掌握AQS的多么重要！！！

```java
    private void doReleaseShared() {

        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```

### 总结

如果掌握前面的知识，Semaphore还是很容易理解的。还有就是理解Semaphore的应用场景，通常跟我们的资源类一起使用。