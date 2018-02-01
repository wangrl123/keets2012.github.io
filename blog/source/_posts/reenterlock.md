---
title: 并发Lock之ReentrantLock实现原理
date: 2017-5-22
categories: 并发编程
tags:
- java
---
我们在之前介绍了[并发编程的锁机制：synchronized和lock](http://blueskykong.com/2017/05/14/lock/)，lock接口的重要实现类是可重入锁`ReentrantLock`。而上一篇[并发Lock之AQS（AbstractQueuedSynchronizer）详解](http://blueskykong.com/2017/05/18/aqs/)介绍了AQS，谈到ReentrantLock，不得不谈抽象类AbstractQueuedSynchronizer（AQS）。AQS定义了一套多线程访问共享资源的同步器框架，ReentrantLock的实现依赖于该同步器。本文在介绍过AQS，结合其具体的实现类`ReentrantLock`分析实现原理。

## ReentrantLock类图

![ReentrantLock](http://ovcjgn2x0.bkt.clouddn.com/reenterlock-class.png "ReentrantLock类图")

ReentrantLock实现了Lock接口，内部有三个内部类，Sync、NonfairSync、FairSync，Sync是一个抽象类型，它继承AbstractQueuedSynchronizer，这个AbstractQueuedSynchronizer是一个模板类，它实现了许多和锁相关的功能，并提供了钩子方法供用户实现，比如tryAcquire，tryRelease等。Sync实现了AbstractQueuedSynchronizer的tryRelease方法。NonfairSync和FairSync两个类继承自Sync，实现了lock方法，公平抢占和非公平抢占针对tryAcquire有不同的实现。本文重点介绍ReentrantLock默认的实现，即非公平锁的获取锁和释放锁的实现。

## 非公平锁的lock方法

### lock方法
- 在初始化ReentrantLock的时候，如果我们不传参数，那么默认使用非公平锁，也就是NonfairSync。

```java
public ReentrantLock() {   
  sync = new NonfairSync();  
} 
```

- 当我们调用ReentrantLock的lock方法的时候，实际上是调用了NonfairSync的lock方法，这个方法先用CAS操作，去尝试抢占该锁。如果成功，就把当前线程设置在这个锁上，表示抢占成功。如果失败，则调用acquire模板方法，等待抢占。代码如下：

```java
static final class NonfairSync extends Sync {
    final void lock() {
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}
```

- 上面调用`acquire(1)`实际上使用的是`AbstractQueuedSynchronizer`的acquire方法，它是一套锁抢占的模板，总体原理是先去尝试获取锁，如果没有获取成功，就在CLH队列中增加一个当前线程的节点，表示等待抢占。然后进入CLH队列的抢占模式，进入的时候也会去执行一次获取锁的操作，如果还是获取不到，就调用LockSupport.park将当前线程挂起。那么当前线程什么时候会被唤醒呢？当持有锁的那个线程调用unlock的时候，会将CLH队列的头节点的下一个节点上的线程唤醒，调用的是LockSupport.unpark方法。acquire代码比较简单，具体如下：

```java
public final void acquire(int arg) {
	if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
	    selfInterrupt();
}
```

- acquire方法内部先使用tryAcquire这个钩子方法去尝试再次获取锁，这个方法在NonfairSync这个类中其实就是使用了nonfairTryAcquire，具体实现原理是先比较当前锁的状态是否是0，如果是0，则尝试去原子抢占这个锁（设置状态为1，然后把当前线程设置成独占线程），如果当前锁的状态不是0，就去比较当前线程和占用锁的线程是不是一个线程，如果是，会去增加状态变量的值，从这里看出可重入锁之所以可重入，就是同一个线程可以反复使用它占用的锁。如果以上两种情况都不通过，则返回失败false。代码如下：

```java
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

- tryAcquire一旦返回false，就会则进入acquireQueued流程，也就是基于CLH队列的抢占模式

首先，在CLH锁队列尾部增加一个等待节点，这个节点保存了当前线程，通过调用addWaiter实现，这里需要考虑初始化的情况，在第一个等待节点进入的时候，需要初始化一个头节点然后把当前节点加入到尾部，后续则直接在尾部加入节点就行了。

```java
private Node addWaiter(Node mode) {
	// 初始化一个节点，这个节点保存当前线程
    Node node = new Node(Thread.currentThread(), mode);
    // 当CLH队列不为空的视乎，直接在队列尾部插入一个节点
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
	// 当CLH队列为空的时候，调用enq方法初始化队列
    enq(node);
    return node;
}

private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // 初始化节点，头尾都指向一个空节点
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {// 考虑并发初始化
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

- 将节点增加到CLH队列后，进入acquireQueued方法。

首先，外层是一个无限for循环，如果当前节点是头节点的下个节点，并且通过tryAcquire获取到了锁，说明头节点已经释放了锁，当前线程是被头节点那个线程唤醒的，这时候就可以将当前节点设置成头节点，并且将failed标记设置成false，然后返回。至于上一个节点，它的next变量被设置为null，在下次GC的时候会清理掉。

如果本次循环没有获取到锁，就进入线程挂起阶段，也就是shouldParkAfterFailedAcquire这个方法。

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
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

- 如果尝试获取锁失败，就会进入shouldParkAfterFailedAcquire方法，会判断当前线程是否挂起，如果前一个节点已经是SIGNAL状态，则当前线程需要挂起。如果前一个节点是取消状态，则需要将取消节点从队列移除。如果前一个节点状态是其他状态，则尝试设置成SIGNAL状态，并返回不需要挂起，从而进行第二次抢占。完成上面的事后进入挂起阶段。

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        //
        return true;
    if (ws > 0) {
        //
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        //
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

- 当进入挂起阶段，会进入parkAndCheckInterrupt方法，则会调用LockSupport.park(this)将当前线程挂起。

```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}

```
## 非公平锁的unlock方法
- 调用unlock方法，其实是直接调用AbstractQueuedSynchronizer的release操作。

```java
public void unlock() {
	sync.release(1);
}
```

- 进入release方法，内部先尝试tryRelease操作,主要是去除锁的独占线程，然后将状态减一，这里减一主要是考虑到可重入锁可能自身会多次占用锁，只有当状态变成0，才表示完全释放了锁。

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

- 一旦tryRelease成功，则将CHL队列的头节点的状态设置为0，然后唤醒下一个非取消的节点线程。

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
          free = true;
          setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
 }
```

- 一旦下一个节点的线程被唤醒，被唤醒的线程就会进入acquireQueued代码流程中，去获取锁。

```java
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null) 
       LockSupport.unpark(s.thread);
}
```

## tryLock
 在ReetrantLock的`tryLock(long timeout, TimeUnit unit) `提供了超时获取锁的功能。它的语义是在指定的时间内如果获取到锁就返回true，获取不到则返回false。这种机制避免了线程无限期的等待锁释放。
 
 ```java
 public boolean tryLock(long timeout, TimeUnit unit)
        throws InterruptedException {
    return sync.tryAcquireNanos(1, unit.toNanos(timeout));
}
 ```
 具体看一下内部类里面的方法`tryAcquireNanos`：
 
 ```java
 public final boolean tryAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquire(arg) ||
        doAcquireNanos(arg, nanosTimeout);
}
```
如果线程被中断了，那么直接抛出InterruptedException。如果未中断，先尝试获取锁，获取成功就直接返回，获取失败则进入doAcquireNanos。tryAcquire我们已经看过，这里重点看一下doAcquireNanos做了什么。

```java
private boolean doAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    // 起始时间
    long lastTime = System.nanoTime();
    // 线程入队
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        // 又是自旋!
        for (;;) {
            // 获取前驱节点
            final Node p = node.predecessor();
            // 如果前驱是头节点并且占用锁成功,则将当前节点变成头结点
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return true;
            }
            // 如果已经超时,返回false
            if (nanosTimeout <= 0)
                return false;
            // 超时时间未到,且需要挂起
            if (shouldParkAfterFailedAcquire(p, node) &&
                    nanosTimeout > spinForTimeoutThreshold)
                // 阻塞当前线程直到超时时间到期
                LockSupport.parkNanos(this, nanosTimeout);
            long now = System.nanoTime();
            // 更新nanosTimeout
            nanosTimeout -= now - lastTime;
            lastTime = now;
            if (Thread.interrupted())
                //相应中断
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

doAcquireNanos的流程简述为：线程先入等待队列，然后开始自旋，尝试获取锁，获取成功就返回，失败则在队列里找一个安全点把自己挂起直到超时时间过期。这里为什么还需要循环呢？因为当前线程节点的前驱状态可能不是SIGNAL，那么在当前这一轮循环中线程不会被挂起，然后更新超时时间，开始新一轮的尝试。

## 总结
ReentrantLock是可重入的锁，其内部使用的就是独占模式的AQS。公平锁和非公平锁不同之处在于，公平锁在获取锁的时候，不会先去检查state状态，而是直接执行aqcuire(1)。公平锁多了hasQueuePredecessors这个方法，这个方法用于判断CHL队列中是否有节点，对于公平锁，如果CHL队列有节点，则新进入竞争的线程一定要在CHL上排队，而非公平锁则是无视CHL队列中的节点，直接进行竞争抢占，这就有可能导致CHL队列上的节点永远获取不到锁，这就是非公平锁之所以不公平的原因，这里不再赘述。 


### 参考
1. [Java中可重入锁ReentrantLock原理剖析](http://blog.csdn.net/albertfly/article/details/52403508)
2. [ReentrantLock实现原理](http://blog.csdn.net/u011202334/article/details/73188404)
