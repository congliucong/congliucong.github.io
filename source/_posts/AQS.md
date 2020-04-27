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


#### addWaiter
 把当前线程创建为一个Node节点，然后入队。
```java
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        //先判断尾结点存不存在
        if (pred != null) {
            node.prev = pred;
            //存在的话，用CAS将tail指向node
            if (compareAndSetTail(pred, node)) {
                //将当前节点入队，置于对尾。
                pred.next = node;
                return node;
            }
        }
        //如果失败或者尾结点不存在，则不断尝试。
        enq(node);
        return node;
    }
```
当初看到这里时，不太明白，compareAndSetTail(pred, node)这里不是将pred指向node了吗?为什么下面还要pred.next = node;
后来发现是自己错了。
```java
>    private final boolean compareAndSetTail(Node expect, Node update) {
>        return unsafe.compareAndSwapObject(this, tailOffset, expect, update);
>   }
```
这个pred是expect值，这个compareAndSetTail意思是，判断偏移量为tailOffset的值是否等于expect，如果是则将这里的值替换为update。位于tailOffset的值是tail。
因此compareAndSetTail(pred, node)这里是将tail指向node，然后pred.next = node;是将pred指向node，完成 node 的入队，此时 node 位于对尾。 妙啊！ 

#### enq


```java
    private Node enq(final Node node) {
        //不断循环，直到成功。
        for (;;) {
            Node t = tail;
            //如果尾结点为空，说明队列还没初始化，先创建一个空的头结点
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {//用CAS将当前节点入队
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```
入队过程图示如下：

* 通过当前线程创建节点。
* pred指向尾结点tail
* 如果尾结点为空，先初始化一个头结点，头结点是个无参构造的头节点
* CAS将tail指向当前节点
* 最后pred的next指针指向当前节点完成入队。
![](AQS/enq.jpg)

#### acquireQueued

```java
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            //中断标记位
            boolean interrupted = false;
            for (;;) {
                //当前节点的前驱节点
                final Node p = node.predecessor();
                //如果前驱节点为头结点，那么就去尝试获取锁
                if (p == head && tryAcquire(arg)) {
                    //获取锁成功后，将自己设置为头结点
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                //如果前驱节点不为头结点，那么就去判断是否需要中断。
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
我们来看看什么情况下需要中断？
shouldParkAfterFailedAcquire()----true---> parkAndCheckInterrupt()

```java
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;

        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```

1. 如果当前节点已经被设置为signal了，标识当前节点处于等待状态,那么可以return true
2. 如果前驱节点>0，说明cancelled，说明该节点已经超时或者中断，那么需要从同步队列中取消，一直往前找，直到找到不是取消的节点。然后返回false。
3. 走到compareAndSetWaitStatus(pred, ws, Node.SIGNAL);说明前驱节点不是取消，那么肯定就是0 or PROPAGATE，那么用CAS将前驱节点设置为SINNAL，返回true.

可以看出，当shouldParkAfterFailedAcquire返回true时，执行parkAndCheckInterrupt方法， 即只有前驱节点是SINNAL时，才需要当前节点PARK。

```java
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```
将当前线程阻塞，然后返回当前线程的中断状态true。
这个地方又思考了半天才想明白。
首先这里要分两种情况来讲，如果当前线程为非中断状态，那么执行到LockSupport.park(this);线程就会阻塞在这里，等待其他线程释放锁后唤醒。
如果当前线程处于中断状态，那么执行到LockSupport.park(this);后，会***继续执行***，然后执行***Thread.interrupted()***; 然后我就疑问，那这不是死循环了吗？一直抢锁失败，然后又阻塞不了，不就死循环了。。。
我天真了，不，确切说是我太菜了。都是满满的知识点。

```java
    public static boolean interrupted() {
        return currentThread().isInterrupted(true);
    }

        /**
     * Tests if some Thread has been interrupted.  The interrupted state
     * is reset or not based on the value of ClearInterrupted that is
     * passed.
     */
    private native boolean isInterrupted(boolean ClearInterrupted);
```
看注释，interrupted这个方法返回中断状态，同时***重置***了中断状态。
所以，如果当前线程是中断状态，当第一次进来时，park方法是无效的，此时会返回true，并且重置中断状态为false，由于外面是个死循环，所以，再次进来时，此时线程已经不是中断状态了，所以执行park的时候就会成功阻塞。
因此这样不管线程是中断还是非中断状态，都不会死循环，导致cpu飙高。妙啊！！

#### cancelAcquire(node)

cancelAcquire是在acquireQueued方法中的Finally代码，当出现异常是，会进入这个方法，将该节点置为CANCELLED状态。
```java
    private void cancelAcquire(Node node) {
        // 当节点不存在，忽略。
        if (node == null)
            return;
        //把该节点不再关联任何线程
        node.thread = null;

        // 从后向前找，跳过取消状态的节点。
        Node pred = node.prev;
        while (pred.waitStatus > 0)
            node.prev = pred = pred.prev;

        // 获取过滤的前驱节点的后继节点
        Node predNext = pred.next;

        // 把当前节点状态设置为取消
        node.waitStatus = Node.CANCELLED;

        // 如果当前节点是尾结点，则把从后往前第一个非取消的节点设置为尾结点
        // 因为自己是尾结点，所有expect= node,upadte= pred.
        if (node == tail && compareAndSetTail(node, pred)) {
            // 成功后，将尾结点的next设置为null.
            compareAndSetNext(pred, predNext, null);
        } else {
            int ws;
            // 如果当前节点不是头结点的后继节点 //1.判断当前节点的前驱节点是否为SIGNAL.
            //2.如果不是，把前驱节点设置为SIGNAL看是否成功。
            //如果1 2 中有一个为true,在判断当前节点线程是否为null
            //如果上面条件都满足，则把当前节点的前驱节点的后继节点指向当前节点的后继节点
            if (pred != head &&
                ((ws = pred.waitStatus) == Node.SIGNAL ||
                 (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
                pred.thread != null) {
                Node next = node.next;
                if (next != null && next.waitStatus <= 0)
                    compareAndSetNext(pred, predNext, next);
            } else {
                //如果当前节点是head的后继节点，或者上述条件不满足
                //那么就唤醒当前节点的后继节点
                unparkSuccessor(node);
            }

            node.next = node; // help GC
        }
    }
```
这个方法有点复杂。我也是读了好多遍才搞懂。
首先，找到当前节点的前驱节点，如果是CANCELLED状态，就一直往前找，找到不是CANCELLED状态未知，然后把自己和前驱节点建立联系。然后将自己设置为CANCELLED状态。
然后，1.如果自己是尾结点，则把tail向前移动一位，把前驱节点的next设为null,自己就出队了。但是此时自己的prev还指向前一个节点。
2.如果当前节点既不是head节点的后继节点，也不是尾结点，就把当前节点的前驱节点的next指向当前节点的next.然后自己的next指向自己。
3.如果当前节点是head的后继节点，则唤醒后继节点。自己的next指向自己。

这里有个问题，既然这里时将自己设置为CANCELLED，然后出队。那么为啥只改next指针，而不改prev指针呢？还有什么时候修改prev指针呢？
在执行cancelAcquire方法时，当前节点的前驱节点可能已经移除出队了。如果这里修改自己的prev，有可能指向一个已经出队的节点。，而在shouldParkAfterFailedAcquire方法中，已经从后往前，将CANCELLED的节点给去掉了。那为啥在shouldParkAfterFailedAcquire里面可以修改prev指针呢，因为进入这个方法的前提条件是获取锁失败，即已经有人获取锁了，所以当前节点的前面节点都是稳定的。而且在cancelAcquire修改next指针是通过CAS修改的，shouldParkAfterFailedAcquire是直接修改的，也可以体现这一点。 魔鬼啊！！

CANCELLED状态的节点的节点只有在获取锁发生异常后将状态置为CANCELLED，shouldParkAfterFailedAcquire方法会将断开CANCELLED节点的prev,然后调用cancelAcquire方法后将自己的next指向自己。一来一回，将CANCELLED状态的节点的前后指针都给修改了，随后该节点会被JVM给回收掉。

因此可以总结 ReentrantLock 加锁过程是：
![](AQS/lock.jpg)

#### ReentrantLock释放锁











































> 参考资料
>
> 1. http://cmsblogs.com/?cat=151&paged=2
> 2. https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html

