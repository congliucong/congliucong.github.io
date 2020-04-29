---
title: Java基础之JUC(六)CyclicBarrier
date: 2020-04-29 15:54:03
tags: Java
categories: Java基础
---

继续JUC之旅，今天来瞅瞅CyclicBarrier。光看名字，有两重含义：1.循环 2.栅栏。所以CyclicBarrier适合于那些多个线程互相等待，等到都到栅栏处了，再一起放行，并且这个栅栏还可以循环利用。

<!-- more -->

### CyclicBarrier原理

CyclicBarrier跟我们之前分析的CountDownLatch、Semaphore不同的是，CyclicBarrier内部没有AQS的子类，而是使用ReentrantLock、Condition来实现。

1. parties代表一共有多少个线程。
2. barrierCommand表示等所有线程到达栅栏后执行的命令
3. generation用来控制屏障的循环使用
4. count表示还有多少个线程在等待。

```java
    /** The lock for guarding barrier entry */
    private final ReentrantLock lock = new ReentrantLock();
    /** Condition to wait on until tripped */
    private final Condition trip = lock.newCondition();
    /** The number of parties */
    private final int parties;
    /* The command to run when tripped */
    private final Runnable barrierCommand;
    /** The current generation */
    private Generation generation = new Generation();

    /**
     * Number of parties still waiting. Counts down from parties to 0
     * on each generation.  It is reset to parties on each new
     * generation or when broken.
     */
    private int count;
```

构造函数：

parties表示需要拦截的线程个数。barrierAction表示CyclicBarrier接收的Runnable命令，用于在线程都达到栅栏时，优先执行。随后再执行线程await()之后的内容。

```java
    public CyclicBarrier(int parties) {
        this(parties, null);
    }

    public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties <= 0) throw new IllegalArgumentException();
        this.parties = parties;
        this.count = parties;
        this.barrierCommand = barrierAction;
    }
```

### CyclicBarrier源码分析

#### await()方法

```java
    public int await() throws InterruptedException, BrokenBarrierException {
        try {
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe); // cannot happen
        }
    }
```

可以看到dowait就是整个CyclicBarrier的核心方法。
```java
    private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        final ReentrantLock lock = this.lock;
        //获取锁
        lock.lock();
        try {

            final Generation g = generation;
            //判断如果generation处于broken状态，则抛出异常
            if (g.broken)
                throw new BrokenBarrierException();
            //如果线程发生中断
            if (Thread.interrupted()) {
                //终止CyclicBarrier，并且唤醒所有线程
                breakBarrier();
                throw new InterruptedException();
            }
            //当线程进来后，将数量减1
            int index = --count;
            //如果index等于0，说明所有线程都到达栅栏
            //触发CyclicBarrier的command任务
            if (index == 0) {  // tripped
                boolean ranAction = false;
                try {
                    final Runnable command = barrierCommand;
                    if (command != null)
                        command.run();
                    ranAction = true;
                    //唤醒所有线程，然后产生新的generation
                    nextGeneration();
                    return 0;
                } finally {
                    if (!ranAction)
                        breakBarrier();
                }
            }

            // 走到这里，说明还有线程没有到达栅栏。
            for (;;) {
                try {
                    //如果没有超时设置，那么执行codition的await方法，进入等待队列
                    if (!timed)
                        trip.await();
                    else if (nanos > 0L)
                        //超时等待
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        // We're about to finish waiting even if we had not
                        // been interrupted, so this interrupt is deemed to
                        // "belong" to subsequent execution.
                        Thread.currentThread().interrupt();
                    }
                }

                if (g.broken)
                    throw new BrokenBarrierException();
                //如果generation不一样，说明已经下一轮了
                if (g != generation)
                    return index;
                //如果设置了超时并且时间已到，那么终止CyclicBarrier
                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            //释放锁
            lock.unlock();
        }
    }
```

我们可以分析出，如果线程不是最后一个执行await，那么该线程会进入等待队列阻塞。除非：

1. 最后一个线程到达，栅栏打开
2. 某个线程超时了，如果超时（需要注意的trip.awaitNanos(nanos)这里返回的时间差值），往下执行到 if (timed && nanos <= 0L)然后执行breakBarrier()抛出TimeoutException()异常。同时唤醒其他线程。
3. 某个线程被打断，则会被catch到。
4. 他的某个线程在此barrier调用reset()方法。reset()方法用于将屏障重置为初始状态

我们最后分析下，如果线程被中断，抛出异常被catch后：

```java
catch (InterruptedException ie) {
    if (g == generation && ! g.broken) {
        breakBarrier();
        throw ie;
    } else {
        // We're about to finish waiting even if we had not
        // been interrupted, so this interrupt is deemed to
        // "belong" to subsequent execution.
        Thread.currentThread().interrupt();
    }
}
```

什么时候这个if (g == generation && ! g.broken)条件成立？
