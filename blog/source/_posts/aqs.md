---
title: 并发Lock之AQS（AbstractQueuedSynchronizer）详解
date: 2017-5-18 
categories: 并发编程
tags:
- java
---
## 1. J.U.C的lock包结构
上一篇文章讲了[并发编程的锁机制：synchronized和lock](http://blueskykong.com/2017/05/14/lock/)，主要介绍了Java并发编程中常用的锁机制。Lock是一个接口，而synchronized是Java中的关键字，synchronized是基于jvm实现。Lock锁可以被中断，支持定时锁等。Lock的实现类，可重入锁ReentrantLock，我们有讲到其具体用法。而谈到ReentrantLock，不得不谈抽象类AbstractQueuedSynchronizer（AQS）。抽象的队列式的同步器，AQS定义了一套多线程访问共享资源的同步器框架，许多同步类实现都依赖于它，如常用的ReentrantLock、ThreadPoolExecutor。

![lock](http://ovcjgn2x0.bkt.clouddn.com/lock%E5%8C%85%E7%BB%93%E6%9E%84.jpg "Lock包结构")

## 2. AQS介绍

AQS是一个抽象类，主是是以继承的方式使用。AQS本身是没有实现任何同步接口的，它仅仅只是定义了同步状态的获取和释放的方法来供自定义的同步组件的使用。AQS抽象类包含如下几个方法：

AQS定义两种资源共享方式：Exclusive（独占，只有一个线程能执行，如ReentrantLock）和Share（共享，多个线程可同时执行，如Semaphore/CountDownLatch）。共享模式时只用 Sync Queue, 独占模式有时只用 Sync Queue, 但若涉及 Condition, 则还有 Condition Queue。在子类的 tryAcquire, tryAcquireShared 中实现公平与非公平的区分。

不同的自定义同步器争用共享资源的方式也不同。自定义同步器在实现时只需要实现共享资源state的获取与释放方式即可，至于具体线程等待队列的维护（如获取资源失败入队/唤醒出队等），AQS已经在顶层实现好了。

整个 AQS 分为以下几部分:

- Node 节点, 用于存放获取线程的节点, 存在于 Sync Queue, Condition Queue, 这些节点主要的区分在于 waitStatus 的值(下面会详细叙述)
- Condition Queue, 这个队列是用于独占模式中, 只有用到 Condition.awaitXX 时才会将 node加到 tail 上(PS: 在使用 Condition的前提是已经获取 Lock)
- Sync Queue, 独占 共享的模式中均会使用到的存放 Node 的 CLH queue(主要特点是, 队列中总有一个 dummy 节点, 后继节点获取锁的条件由前继节点决定, 前继节点在释放 lock 时会唤醒sleep中的后继节点)
- ConditionObject, 用于独占的模式, 主要是线程释放lock, 加入 Condition Queue, 并进行相应的 signal 操作
- 独占的获取lock (acquire, release), 例如 ReentrantLock。
- 共享的获取lock (acquireShared, releaseShared), 例如 ReeantrantReadWriteLock, Semaphore, CountDownLatch

下面我们具体来分析一下AQS实现的源码。
## 3. 内部类 Node
Node 节点是代表获取lock的线程, 存在于 Condition Queue, Sync Queue 里面， 而其主要就是 nextWaiter (标记共享还是独占),waitStatus 标记node的状态。

![node](https://upload-images.jianshu.io/upload_images/263562-328067a54dfdb18f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/134 "内部类 Node")

```java
static final class Node {
    /** 标识节点是否是 共享的节点(这样的节点只存在于 Sync Queue 里面) */
    static final Node SHARED = new Node();
    //独占模式
    static final Node EXCLUSIVE = null;
    /**
     *  CANCELLED 说明节点已经 取消获取 lock 了(一般是由于 interrupt 或 timeout 导致的)
     *  很多时候是在 cancelAcquire 里面进行设置这个标识
     */
    static final int CANCELLED = 1;

    /**
     * SIGNAL 标识当前节点的后继节点需要唤醒(PS: 这个通常是在 独占模式下使用, 在共享模式下有时用 PROPAGATE)
     */
    static final int SIGNAL = -1;
    
    //当前节点在 Condition Queue 里面
    static final int CONDITION = -2;
    
    /**
     * 当前节点获取到 lock 或进行 release lock 时, 共享模式的最终状态是 PROPAGATE(PS: 有可能共享模式的节点变成 PROPAGATE 之前就被其后继节点抢占 head 节点, 而从Sync Queue中被踢出掉)
     */
    static final int PROPAGATE = -3;

    volatile int waitStatus;

    /**
     * 节点在 Sync Queue 里面时的前继节点(主要来进行 skip CANCELLED 的节点)
     * 注意: 根据 addWaiter方法:
     *  1. prev节点在队列里面, 则 prev != null 肯定成立
     *  2. prev != null 成立, 不一定 node 就在 Sync Queue 里面
     */
    volatile Node prev;

    /**
     * Node 在 Sync Queue 里面的后继节点, 主要是在release lock 时进行后继节点的唤醒
     * 而后继节点在前继节点上打上 SIGNAL 标识, 来提醒他 release lock 时需要唤醒
     */
    volatile Node next;

    //获取 lock 的引用
    volatile Thread thread;

    /**
     * 作用分成两种:
     *  1. 在 Sync Queue 里面, nextWaiter用来判断节点是 共享模式, 还是独占模式
     *  2. 在 Condition queue 里面, 节点主要是链接且后继节点 (Condition queue是一个单向的, 不支持并发的 list)
     */
    Node nextWaiter;

    // 当前节点是否是共享模式
    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    // 获取 node 的前继节点
    final Node predecessor() throws NullPointerException{
        Node p = prev;
        if(p == null){
            throw new NullPointerException();
        }else{
            return p;
        }
    }

    Node(){
        // Used to establish initial head or SHARED marker
    }

    // 初始化 Node 用于 Sync Queue 里面
    Node(Thread thread, Node mode){     // Used by addWaiter
        this.nextWaiter = mode;
        this.thread = thread;
    }

    //初始化 Node 用于 Condition Queue 里面
    Node(Thread thread, int waitStatus){ // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
```

waitStatus的状态变化:   

1. 线程刚入 Sync Queue 里面, 发现独占锁被其他人获取, 则将其前继节点标记为 SIGNAL, 然后再尝试获取一下锁(调用 tryAcquire 方法)
2. 若调用 tryAcquire 方法获取失败, 则判断一下是否前继节点被标记为 SIGNAL, 若是的话 直接 block(block前会确保前继节点被标记为SIGNAL, 因为前继节点在进行释放锁时根据是否标记为 SIGNAL 来决定唤醒后继节点与否 <- 这是独占的情况下)
3. 前继节点使用完lock, 进行释放, 因为自己被标记为 SIGNAL, 所以唤醒其后继节点

waitStatus 变化过程:   

1. 独占模式下:  0(初始) -> signal(被后继节点标记为release需要唤醒后继节点) -> 0 (等释放好lock, 会恢复到0)
2. 独占模式 + 使用 Condition情况下: 0(初始) -> signal(被后继节点标记为release需要唤醒后继节点) -> 0 (等释放好lock, 会恢复到0)其上可能涉及 中断与超时, 只是多了一个 CANCELLED, 当节点变成 CANCELLED, 后就等着被清除。
3. 共享模式下: 0(初始) -> PROPAGATE(获取 lock 或release lock 时) (获取 lock 时会调用 setHeadAndPropagate 来进行 传递式的唤醒后继节点, 直到碰到 独占模式的节点)
4. 共享模式 + 独占模式下: 0(初始) -> signal(被后继节点标记为release需要唤醒后继节点) -> 0 (等释放好lock, 会恢复到0)

其上的这些状态变化主要在: doReleaseShared , shouldParkAfterFailedAcquire 里面。

## 4. Condition Queue
Condition Queue 是一个并发不安全的, 只用于独占模式的队列(PS: 为什么是并发不安全的呢? 主要是在操作 Condition 时, 线程必需获取 独占的 lock, 所以不需要考虑并发的安全问题);
而当Node存在于 Condition Queue 里面, 则其只有 waitStatus, thread, nextWaiter 有值, 其他的都是null(其中的 waitStatus 只能是 CONDITION, 0(0 代表node进行转移到 Sync Queue里面, 或被中断/timeout)); 这里有个注意点, 就是当线程被中断或获取 lock 超时, 则一瞬间 node 会存在于 Condition Queue, Sync Queue 两个队列中.

![ConditionQueue](https://upload-images.jianshu.io/upload_images/263562-fd3b56ad835e9f95.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700 "Condition Queue")
节点 Node4, Node5, Node6, Node7 都是调用 Condition.awaitXX 方法加入 Condition Queue(PS: 加入后会将原来的 lock 释放)。

### 4.1 入队列方法 addConditionWaiter
将当前线程封装成一个 Node 节点放入到 Condition Queue 里面大家可以注意到, 下面对 Condition Queue 的操作都没考虑到 并发(Sync Queue 的队列是支持并发操作的), 这是为什么呢? 因为在进行操作 Condition 是当前的线程已经获取了AQS的独占锁, 所以不需要考虑并发的情况。

```java
private Node addConditionWaiter(){
    Node t = lastWaiter;                                
    // Condition queue 的尾节点           
	// 尾节点已经Cancel, 直接进行清除,
    /** 
    * 当Condition进行 awiat 超时或被中断时, Condition里面的节点是没有被删除掉的, 需要其	 * 他await 在将线程加入 Condition Queue 时调用addConditionWaiter而进而删除, 或 await 操作差不多结束时, 调用 "node.nextWaiter != null" 进行判断而删除 (PS: 通过 signal 进行唤
    * 醒时 node.nextWaiter 会被置空, 而中断和超时时不会)
    */
    if(t != null && t.waitStatus != Node.CONDITION){
    	/** 
    	* 调用 unlinkCancelledWaiters 对 "waitStatus != Node.CONDITION" 的节点进行		* 删除(在Condition里面的Node的waitStatus 要么是CONDITION(正常), 要么就是 0 
    	* (signal/timeout/interrupt))
    	*/
        unlinkCancelledWaiters();                     
        t = lastWaiter;                     
    }
    //将线程封装成 node 准备放入 Condition Queue 里面
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    if(t == null){
    	//Condition Queue 是空的
        firstWaiter = node;                           
    } else {
    	// 追加到 queue 尾部
        t.nextWaiter = node;                          
    }
    lastWaiter = node;                               
    return node;
}
```

### 4.2 删除Cancelled节点的方法 unlinkCancelledWaiters
当Node在Condition Queue 中, 若状态不是 CONDITION, 则一定是被中断或超时。在调用 addConditionWaiter 将线程放入 Condition Queue 里面时或 awiat 方法获取结束时 进行清理 Condition queue 里面的因 timeout/interrupt 而还存在的节点。这个删除操作比较巧妙, 其中引入了 trail 节点， 可以理解为traverse整个 Condition Queue 时遇到的最后一个有效的节点。

```java
private void unlinkCancelledWaiters(){
    Node t = firstWaiter;
    Node trail = null;
    while(t != null){
        Node next = t.nextWaiter;               // 1. 先初始化 next 节点
        if(t.waitStatus != Node.CONDITION){   // 2. 节点不有效, 在Condition Queue 里面 Node.waitStatus 只有可能是 CONDITION 或是 0(timeout/interrupt引起的)
            t.nextWaiter = null;               // 3. Node.nextWaiter 置空
            if(trail == null){                  // 4. 一次都没有遇到有效的节点
                firstWaiter = next;            // 5. 将 next 赋值给 firstWaiter(此时 next 可能也是无效的, 这只是一个临时处理)
            } else {
                trail.nextWaiter = next;       // 6. next 赋值给 trail.nextWaiter, 这一步其实就是删除节点 t
            }
            if(next == null){                  // 7. next == null 说明 已经 traverse 完了 Condition Queue
                lastWaiter = trail;
            }
        }else{
            trail = t;                         // 8. 将有效节点赋值给 trail
        }
        t = next;
    }
}
```
### 4.3 转移节点的方法 transferForSignal
transferForSignal只有在节点被正常唤醒才调用的正常转移的方法。   
 将Node 从Condition Queue 转移到 Sync Queue 里面在调用transferForSignal之前, 会 first.nextWaiter = null;而我们发现若节点是因为 timeout / interrupt 进行转移, 则不会进行这步操作; 两种情况的转移都会把 wautStatus 置为 0

```java
final boolean transferForSignal(Node node){
    /**
     * If cannot change waitStatus, the node has been cancelled
     */
    if(!compareAndSetWaitStatus(node, Node.CONDITION, 0)){ // 1. 若 node 已经 cancelled 则失败
        return false;
    }

    Node p = enq(node);                                 // 2. 加入 Sync Queue
    int ws = p.waitStatus;
    if(ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL)){ // 3. 这里的 ws > 0 指Sync Queue 中node 的前继节点cancelled 了, 所以, 唤醒一下 node ; compareAndSetWaitStatus(p, ws, Node.SIGNAL)失败, 则说明 前继节点已经变成 SIGNAL 或 cancelled, 所以也要 唤醒
        LockSupport.unpark(node.thread);
    }
    return true;
}
```

### 4.4 转移节点的方法 transferAfterCancelledWait
transferAfterCancelledWait 在节点获取lock时被中断或获取超时才调用的转移方法。将 Condition Queue 中因 timeout/interrupt 而唤醒的节点进行转移

```java
final boolean transferAfterCancelledWait(Node node){
    if(compareAndSetWaitStatus(node, Node.CONDITION, 0)){ // 1. 没有 node 没有 cancelled , 直接进行转移 (转移后, Sync Queue , Condition Queue 都会存在 node)
        enq(node);
        return true;
    }
    
    while(!isOnSyncQueue(node)){                // 2.这时是其他的线程发送signal,将本线程转移到 Sync Queue 里面的工程中(转移的过程中 waitStatus = 0了, 所以上面的 CAS 操作失败)
        Thread.yield();                         // 这里调用 isOnSyncQueue判断是否已经 入Sync Queue 了
    }
    return false;
}
```

## 5. Sync Queue
AQS内部维护着一个FIFO的CLH队列，所以AQS并不支持基于优先级的同步策略。至于为何要选择CLH队列，主要在于CLH锁相对于MSC锁，他更加容易处理cancel和timeout，同时他具备进出队列快、无所、畅通无阻、检查是否有线程在等待也非常容易（head != tail,头尾指针不同）。当然相对于原始的CLH队列锁，ASQ采用的是一种变种的CLH队列锁：

1. 原始CLH使用的locked自旋，而AQS的CLH则是在每个node里面使用一个状态字段来控制阻塞，而不是自旋。
2. 为了可以处理timeout和cancel操作，每个node维护一个指向前驱的指针。如果一个node的前驱被cancel，这个node可以前向移动使用前驱的状态字段。
3. head结点使用的是傀儡结点。

![SyncQueue](https://upload-images.jianshu.io/upload_images/263562-c520b712aec0a7e4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700 "Sync Queue")

这个图代表有个线程获取lock, 而 Node1, Node2, Node3 则在Sync Queue 里面进行等待获取lock(PS: 注意到 dummy Node 的SINGNAL 这是叫获取 lock 的线程在释放lock时通知后继节点的标示)

### 5.1 Sync Queue 节点入Queue方法
这里有个地方需要注意, 就是初始化 head, tail 的节点, 不一定是 head.next, 因为期间可能被其他的线程进行抢占了。将当前的线程封装成 Node 加入到 Sync Queue 里面。

```java
private Node addWaiter(Node mode){
    Node node = new Node(Thread.currentThread(), mode);      // 1. 封装 Node
    Node pred = tail;
    if(pred != null){                           // 2. pred != null -> 队列中已经有节点, 直接 CAS 到尾节点
        node.prev = pred;                       // 3. 先设置 Node.pre = pred (PS: 则当一个 node在Sync Queue里面时  node.prev 一定 != null(除 dummy node), 但是 node.prev != null 不能说明其在 Sync Queue 里面, 因为现在的CAS可能失败 )
        if(compareAndSetTail(pred, node)){      // 4. CAS node 到 tail
            pred.next = node;                  // 5. CAS 成功, 将 pred.next = node (PS: 说明 node.next != null -> 则 node 一定在 Sync Queue, 但若 node 在Sync Queue 里面不一定 node.next != null)
            return node;
        }
    }
    enq(node);                                 // 6. 队列为空, 调用 enq 入队列
    return node;
}


/**
 * 这个插入会检测head tail 的初始化, 必要的话会初始化一个 dummy 节点, 这个和 ConcurrentLinkedQueue 一样的
 * 将节点 node 加入队列
 * 这里有个注意点
 * 情况:
 *      1. 首先 queue是空的
 *      2. 初始化一个 dummy 节点
 *      3. 这时再在tail后面添加节点(这一步可能失败, 可能发生竞争被其他的线程抢占)
 *  这里为什么要加入一个 dummy 节点呢?
 *      这里的 Sync Queue 是CLH lock的一个变种, 线程节点 node 能否获取lock的判断通过其前继节点
 *      而且这里在当前节点想获取lock时通常给前继节点 打上 signal 的标识(表示前继节点释放lock需要通知我来获取lock)
 *      若这里不清楚的同学, 请先看看 CLH lock的资料 (这是理解 AQS 的基础)
 */
private Node enq(final Node node){
    for(;;){
        Node t = tail;
        if(t == null){ // Must initialize       // 1. 队列为空 初始化一个 dummy 节点 其实和 ConcurrentLinkedQueue 一样
            if(compareAndSetHead(new Node())){  // 2. 初始化 head 与 tail (这个CAS成功后, head 就有值了, 详情将 Unsafe 操作)
                tail = head;
            }
        }else{
            node.prev = t;                      // 3. 先设置 Node.pre = pred (PS: 则当一个 node在Sync Queue里面时  node.prev 一定 != null, 但是 node.prev != null 不能说明其在 Sync Queue 里面, 因为现在的CAS可能失败 )
            if(compareAndSetTail(t, node)){     // 4. CAS node 到 tail
                t.next = node;                  // 5. CAS 成功, 将 pred.next = node (PS: 说明 node.next != null -> 则 node 一定在 Sync Queue, 但若 node 在Sync Queue 里面不一定 node.next != null)
                return t;
            }
        }
    }
}
```

### 5.2 Sync Queue 节点出Queue方法
这里的出Queue的方法其实有两个：
新节点获取lock, 调用setHead抢占head, 并且剔除原head；节点因被中断或获取超时而进行 cancelled, 最后被剔除。

```java
/**
 * 设置 head 节点(在独占模式没有并发的可能, 当共享的模式有可能)
 */
private void setHead(Node node){
    head = node;
    node.thread = null; // 清除线程引用
    node.prev = null; // 清除原来 head 的引用 <- 都是 help GC
}

// 清除因中断/超时而放弃获取lock的线程节点(此时节点在 Sync Queue 里面)
private void cancelAcquire(Node node) {
    if (node == null)
        return;

    node.thread = null;                 // 1. 线程引用清空

    Node pred = node.prev;
    while (pred.waitStatus > 0)       // 2.  若前继节点是 CANCELLED 的, 则也一并清除
        node.prev = pred = pred.prev;
        
    Node predNext = pred.next;         // 3. 这里的 predNext也是需要清除的(只不过在清除时的 CAS 操作需要 它)

    node.waitStatus = Node.CANCELLED; // 4. 标识节点需要清除

    // If we are the tail, remove ourselves.
    if (node == tail && compareAndSetTail(node, pred)) { // 5. 若需要清除额节点是尾节点, 则直接 CAS pred为尾节点
        compareAndSetNext(pred, predNext, null);    // 6. 删除节点predNext
    } else {
        int ws;
        if (pred != head &&
                ((ws = pred.waitStatus) == Node.SIGNAL || // 7. 后继节点需要唤醒(但这里的后继节点predNext已经 CANCELLED 了)
                        (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) && // 8. 将 pred 标识为 SIGNAL
                pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0) // 8. next.waitStatus <= 0 表示 next 是个一个想要获取lock的节点
                compareAndSetNext(pred, predNext, next);
        } else {
            unparkSuccessor(node); // 若 pred 是头节点, 则此刻可能有节点刚刚进入 queue ,所以进行一下唤醒
        }

        node.next = node; // help GC
    }
}
```
## 6. 独占Lock

### 6.1 独占方式获取lock主要流程   

1. 调用 tryAcquire 尝试性的获取锁(一般都是由子类实现), 成功的话直接返回   
2. tryAcquire 调用获取失败, 将当前的线程封装成 Node 加入到 Sync Queue 里面(调用addWaiter), 等待获取 signal 信号   
3. 调用 acquireQueued 进行自旋的方式获取锁(有可能会 repeatedly blocking and unblocking)   
4. 根据acquireQueued的返回值判断在获取lock的过程中是否被中断, 若被中断, 则自己再中断一下(selfInterrupt), 若是响应中断的则直接抛出异常

### 6.2 独占方式获取lock主要分成3类

1. acquire 不响应中断的获取lock, 这里的不响应中断指的是线程被中断后会被唤醒, 并且继续获取lock,在方法返回时, 根据刚才的获取过程是否被中断来决定是否要自己中断一下(方法 selfInterrupt)
2. doAcquireInterruptibly 响应中断的获取 lock, 这里的响应中断, 指在线程获取 lock 过程中若被中断, 则直接抛出异常
3. doAcquireNanos 响应中断及超时的获取 lock, 当线程被中断, 或获取超时, 则直接抛出异常, 获取失败

### 6.3 独占的获取lock 方法 acquire
acquire(int arg)：以独占模式获取对象，忽略中断。

```java
public final void acquire(int arg){
    if(!tryAcquire(arg)&&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) {
        selfInterrupt();
    }
}
```

1. 调用 tryAcquire 尝试性的获取锁(一般都是又子类实现), 成功的话直接返回
2. tryAcquire 调用获取失败, 将当前的线程封装成 Node 加入到 Sync Queue 里面(调用addWaiter), 等待获取 signal 信号
3. 调用 acquireQueued 进行自旋的方式获取锁(有可能会 repeatedly blocking and unblocking) 
4. 根据acquireQueued的返回值判断在获取lock的过程中是否被中断, 若被中断, 则自己再中断一下(selfInterrupt)。

### 6.4 循环获取lock 方法 acquireQueued

```java
final boolean acquireQueued(final Node node, int arg){
        boolean failed = true;
        try {
            boolean interrupted = false;
            for(;;){
                final Node p = node.predecessor();      // 1. 获取当前节点的前继节点 (当一个n在 Sync Queue 里面, 并且没有获取 lock 的 node 的前继节点不可能是 null)
                if(p == head && tryAcquire(arg)){       // 2. 判断前继节点是否是head节点(前继节点是head, 存在两种情况 (1) 前继节点现在占用 lock (2)前继节点是个空节点, 已经释放 lock, node 现在有机会获取 lock); 则再次调用 tryAcquire尝试获取一下
                    setHead(node);                       // 3. 获取 lock 成功, 直接设置 新head(原来的head可能就直接被回收)
                    p.next = null; // help GC          // help gc
                    failed = false;
                    return interrupted;                // 4. 返回在整个获取的过程中是否被中断过 ; 但这又有什么用呢? 若整个过程中被中断过, 则最后我在 自我中断一下 (selfInterrupt), 因为外面的函数可能需要知道整个过程是否被中断过
                }
                if(shouldParkAfterFailedAcquire(p, node) && // 5. 调用 shouldParkAfterFailedAcquire 判断是否需要中断(这里可能会一开始 返回 false, 但在此进去后直接返回 true(主要和前继节点的状态是否是 signal))
                        parkAndCheckInterrupt()){      // 6. 现在lock还是被其他线程占用 那就睡一会, 返回值判断是否这次线程的唤醒是被中断唤醒
                    interrupted = true;
                }
            }
        }finally {
            if(failed){                             // 7. 在整个获取中出错
                cancelAcquire(node);                // 8. 清除 node 节点(清除的过程是先给 node 打上 CANCELLED标志, 然后再删除)
            }
        }
    }
```

主逻辑:

1. 当前节点的前继节点是head节点时，先 tryAcquire获取一下锁, 成功的话设置新 head, 返回
2. 第一步不成功, 检测是否需要sleep, 需要的话就sleep, 等待前继节点在释放lock时唤醒或通过中断来唤醒
3. 整个过程可能需要blocking nonblocking 几次

### 6.5 支持中断获取lock 方法 doAcquireInterruptibly

```java
private void doAcquireInterruptibly(int arg) throws InterruptedException{
    final Node node = addWaiter(Node.EXCLUSIVE);  // 1. 将当前的线程封装成 Node 加入到 Sync Queue 里面
    boolean failed = true;
    try {
        for(;;){
            final Node p = node.predecessor(); // 2. 获取当前节点的前继节点 (当一个n在 Sync Queue 里面, 并且没有获取 lock 的 node 的前继节点不可能是 null)
            if(p == head && tryAcquire(arg)){  // 3. 判断前继节点是否是head节点(前继节点是head, 存在两种情况 (1) 前继节点现在占用 lock (2)前继节点是个空节点, 已经释放 lock, node 现在有机会获取 lock); 则再次调用 tryAcquire尝试获取一下
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }

            if(shouldParkAfterFailedAcquire(p, node) && // 4. 调用 shouldParkAfterFailedAcquire 判断是否需要中断(这里可能会一开始 返回 false, 但在此进去后直接返回 true(主要和前继节点的状态是否是 signal))
                    parkAndCheckInterrupt()){           // 5. 现在lock还是被其他线程占用 那就睡一会, 返回值判断是否这次线程的唤醒是被中断唤醒
                throw new InterruptedException();       // 6. 线程此时唤醒是通过线程中断, 则直接抛异常
            }
        }
    }finally {
        if(failed){                 // 7. 在整个获取中出错(比如线程中断)
            cancelAcquire(node);    // 8. 清除 node 节点(清除的过程是先给 node 打上 CANCELLED标志, 然后再删除)
        }
    }
}
```

acquireInterruptibly(int arg)： 以独占模式获取对象，如果被中断则中止。

```java
public final void acquireInterruptibly(int arg) throws InterruptedException {    
        if (Thread.interrupted())    
            throw new InterruptedException();    
        if (!tryAcquire(arg))       
            doAcquireInterruptibly(arg);     
    }
```

通过先检查中断的状态，然后至少调用一次tryAcquire，返回成功。否则，线程在排队，不停地阻塞与唤醒，调用tryAcquire直到成功或者被中断。

### 6.6 超时&中断获取lock 方法 
tryAcquireNanos(int arg, long nanosTimeout):独占且支持超时模式获取： 带有超时时间，如果经过超时时间则会退出。

```java
private boolean doAcquireNanos(int arg, long nanosTimeout) throws InterruptedException{
    if(nanosTimeout <= 0L){
        return false;
    }

    final long deadline = System.nanoTime() + nanosTimeout; // 0. 计算截至时间
    final Node node = addWaiter(Node.EXCLUSIVE);  // 1. 将当前的线程封装成 Node 加入到 Sync Queue 里面
    boolean failed = true;

    try {
        for(;;){
            final Node p = node.predecessor(); // 2. 获取当前节点的前继节点 (当一个n在 Sync Queue 里面, 并且没有获取 lock 的 node 的前继节点不可能是 null)
            if(p == head && tryAcquire(arg)){  // 3. 判断前继节点是否是head节点(前继节点是head, 存在两种情况 (1) 前继节点现在占用 lock (2)前继节点是个空节点, 已经释放 lock, node 现在有机会获取 lock); 则再次调用 tryAcquire尝试获取一下
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return true;
            }

            nanosTimeout = deadline - System.nanoTime(); // 4. 计算还剩余的时间
            if(nanosTimeout <= 0L){                      // 5. 时间超时, 直接返回
                return false;
            }
            if(shouldParkAfterFailedAcquire(p, node) && // 6. 调用 shouldParkAfterFailedAcquire 判断是否需要中断(这里可能会一开始 返回 false, 但在此进去后直接返回 true(主要和前继节点的状态是否是 signal))
                    nanosTimeout > spinForTimeoutThreshold){ // 7. 若没超时, 并且大于spinForTimeoutThreshold, 则线程 sleep(小于spinForTimeoutThreshold, 则直接自旋, 因为效率更高 调用 LockSupport 是需要开销的)
                LockSupport.parkNanos(this, nanosTimeout);
            }
            if(Thread.interrupted()){                           // 8. 线程此时唤醒是通过线程中断, 则直接抛异常
                throw new InterruptedException();
            }
        }
    }finally {
        if(failed){                 // 9. 在整个获取中出错(比如线程中断/超时)
            cancelAcquire(node);    // 10. 清除 node 节点(清除的过程是先给 node 打上 CANCELLED标志, 然后再删除)
        }
    }
}
```

尝试以独占模式获取，如果中断和超时则放弃。实现时先检查中断的状态，然后至少调用一次tryAcquire。

```java
public final boolean tryAcquireNanos(int arg, long nanosTimeout) throws InterruptedException {    
     if (Thread.interrupted())    
         throw new InterruptedException();    
     return tryAcquire(arg)|| doAcquireNanos(arg, nanosTimeout);    
}
```
### 6.7 释放lock方法

释放 lock 流程：

- 调用子类的 tryRelease 方法释放获取的资源
- 判断是否完全释放lock(这里有 lock 重复获取的情况)
- 判断是否有后继节点需要唤醒, 需要的话调用unparkSuccessor进行唤醒

```java
public final boolean release(int arg){
    if(tryRelease(arg)){   // 1. 调用子类, 若完全释放好, 则返回true(这里有lock重复获取)
        Node h = head;
        if(h != null && h.waitStatus != 0){ // 2. h.waitStatus !=0 其实就是 h.waitStatus < 0 后继节点需要唤醒
            unparkSuccessor(h);   // 3. 唤醒后继节点
        }
        return true;
    }
    return false;
}

/**
 * 唤醒 node 的后继节点
 * 这里有个注意点: 唤醒时会将当前node的标识归位为 0
 * 等于当前节点标识位 的流转过程: 0(刚加入queue) -> signal (被后继节点要求在释放时需要唤醒) -> 0 (进行唤醒后继节点)
 */
private void unparkSuccessor(Node node) {
    logger.info("unparkSuccessor node:" + node + Thread.currentThread().getName());
    
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);       // 1. 清除前继节点的标识
    Node s = node.next;
    logger.info("unparkSuccessor s:" + node + Thread.currentThread().getName());
    if (s == null || s.waitStatus > 0) {         // 2. 这里若在 Sync Queue 里面存在想要获取 lock 的节点,则一定需要唤醒一下(跳过取消的节点)　（PS: s == null发生在共享模式的竞争释放资源）
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)              // 3. 找到 queue 里面最前面想要获取 Lock 的节点
                s = t;
    }
    logger.info("unparkSuccessor s:"+s);
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

## 7. 共享Lock

### 7.1 共享方式获取lock流程
1. 调用 tryAcquireShared 尝试性的获取锁(一般都是由子类实现), 成功的话直接返回
2. tryAcquireShared 调用获取失败, 将当前的线程封装成 Node 加入到 Sync Queue 里面(调用addWaiter), 等待获取 signal 信号
3. 在 Sync Queue 里面进行自旋的方式获取锁(有可能会 repeatedly blocking and unblocking
4. 当获取失败, 则判断是否可以 block(block的前提是前继节点被打上 SIGNAL 标示)
5. 共享与独占获取lock的区别主要在于 在共享方式下获取 lock 成功会判断是否需要继续唤醒下面的继续获取共享lock的节点(及方法 doReleaseShared)

### 7.2 共享方式获取lock主要分成3类
1. acquireShared 不响应中断的获取lock, 这里的不响应中断指的是线程被中断后会被唤醒, 并且继续获取lock,在方法返回时, 根据刚才的获取过程是否被中断来决定是否要自己中断一下(方法 selfInterrupt)
2. doAcquireSharedInterruptibly 响应中断的获取 lock, 这里的响应中断, 指在线程获取 lock 过程中若被中断, 则直接抛出异常
3. doAcquireSharedNanos 响应中断及超时的获取 lock, 当线程被中断, 或获取超时, 则直接抛出异常, 获取失败

### 7.3 获取共享lock 方法 acquireShared

```java
public final void acquireShared(int arg){
    if(tryAcquireShared(arg) < 0){  // 1. 调用子类, 获取共享 lock  返回 < 0, 表示失败
        doAcquireShared(arg);       // 2. 调用 doAcquireShared 当前 线程加入 Sync Queue 里面, 等待获取 lock
    }
}
```

### 7.4 获取共享lock 方法 doAcquireShared

```java
private void doAcquireShared(int arg){
    final Node node = addWaiter(Node.SHARED);       // 1. 将当前的线程封装成 Node 加入到 Sync Queue 里面
    boolean failed = true;

    try {
        boolean interrupted = false;
        for(;;){
            final Node p = node.predecessor();      // 2. 获取当前节点的前继节点 (当一个n在 Sync Queue 里面, 并且没有获取 lock 的 node 的前继节点不可能是 null)
            if(p == head){
                int r = tryAcquireShared(arg);      // 3. 判断前继节点是否是head节点(前继节点是head, 存在两种情况 (1) 前继节点现在占用 lock (2)前继节点是个空节点, 已经释放 lock, node 现在有机会获取 lock); 则再次调用 tryAcquireShared 尝试获取一下
                if(r >= 0){
                    setHeadAndPropagate(node, r);   // 4. 获取 lock 成功, 设置新的 head, 并唤醒后继获取  readLock 的节点
                    p.next = null; // help GC
                    if(interrupted){               // 5. 在获取 lock 时, 被中断过, 则自己再自我中断一下(外面的函数可能需要这个参数)
                        selfInterrupt();
                    }
                    failed = false;
                    return;
                }
            }

            if(shouldParkAfterFailedAcquire(p, node) && // 6. 调用 shouldParkAfterFailedAcquire 判断是否需要中断(这里可能会一开始 返回 false, 但在此进去后直接返回 true(主要和前继节点的状态是否是 signal))
                    parkAndCheckInterrupt()){           // 7. 现在lock还是被其他线程占用 那就睡一会, 返回值判断是否这次线程的唤醒是被中断唤醒
                interrupted = true;
            }
        }
    }finally {
        if(failed){             // 8. 在整个获取中出错(比如线程中断/超时)
            cancelAcquire(node);  // 9. 清除 node 节点(清除的过程是先给 node 打上 CANCELLED标志, 然后再删除)
        }
    }
}
```

### 7.5 获取共享lock 方法 doAcquireSharedInterruptibly

```java
private void doAcquireSharedInterruptibly(int arg) throws InterruptedException{
    final Node node = addWaiter(Node.SHARED);            // 1. 将当前的线程封装成 Node 加入到 Sync Queue 里面
    boolean failed = true;

    try {
        for(;;){
            final Node p = node.predecessor();          // 2. 获取当前节点的前继节点 (当一个n在 Sync Queue 里面, 并且没有获取 lock 的 node 的前继节点不可能是 null)
            if(p == head){
                int r = tryAcquireShared(arg);          // 3. 判断前继节点是否是head节点(前继节点是head, 存在两种情况 (1) 前继节点现在占用 lock (2)前继节点是个空节点, 已经释放 lock, node 现在有机会获取 lock); 则再次调用 tryAcquireShared 尝试获取一下
                if(r >= 0){
                    setHeadAndPropagate(node, r);       // 4. 获取 lock 成功, 设置新的 head, 并唤醒后继获取  readLock 的节点
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }

            if(shouldParkAfterFailedAcquire(p, node) && // 5. 调用 shouldParkAfterFailedAcquire 判断是否需要中断(这里可能会一开始 返回 false, 但在此进去后直接返回 true(主要和前继节点的状态是否是 signal))
                    parkAndCheckInterrupt()){           // 6. 现在lock还是被其他线程占用 那就睡一会, 返回值判断是否这次线程的唤醒是被中断唤醒
                throw new InterruptedException();     // 7. 若此次唤醒是 通过线程中断, 则直接抛出异常
            }
        }
    }finally {
        if(failed){              // 8. 在整个获取中出错(比如线程中断/超时)
            cancelAcquire(node); // 9. 清除 node 节点(清除的过程是先给 node 打上 CANCELLED标志, 然后再删除)
        }
    }
}
```

### 7.6 获取共享lock 方法 doAcquireSharedNanos

```java
private boolean doAcquireSharedNanos(int arg, long nanosTimeout) throws InterruptedException{
    if (nanosTimeout <= 0L){
        return false;
    }

    final long deadline = System.nanoTime() + nanosTimeout;  // 0. 计算超时的时间
    final Node node = addWaiter(Node.SHARED);               // 1. 将当前的线程封装成 Node 加入到 Sync Queue 里面
    boolean failed = true;

    try {
        for(;;){
            final Node p = node.predecessor();          // 2. 获取当前节点的前继节点 (当一个n在 Sync Queue 里面, 并且没有获取 lock 的 node 的前继节点不可能是 null)
            if(p == head){
                int r = tryAcquireShared(arg);          // 3. 判断前继节点是否是head节点(前继节点是head, 存在两种情况 (1) 前继节点现在占用 lock (2)前继节点是个空节点, 已经释放 lock, node 现在有机会获取 lock); 则再次调用 tryAcquireShared 尝试获取一下
                if(r >= 0){
                    setHeadAndPropagate(node, r);       // 4. 获取 lock 成功, 设置新的 head, 并唤醒后继获取  readLock 的节点
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }
            }

            nanosTimeout = deadline - System.nanoTime(); // 5. 计算还剩余的 timeout , 若小于0 则直接return
            if(nanosTimeout <= 0L){
                return false;
            }
            if(shouldParkAfterFailedAcquire(p, node) &&         // 6. 调用 shouldParkAfterFailedAcquire 判断是否需要中断(这里可能会一开始 返回 false, 但在此进去后直接返回 true(主要和前继节点的状态是否是 signal))
                    nanosTimeout > spinForTimeoutThreshold){// 7. 在timeout 小于  spinForTimeoutThreshold 时 spin 的效率, 比 LockSupport 更高
                LockSupport.parkNanos(this, nanosTimeout);
            }
            if(Thread.interrupted()){                           // 7. 若此次唤醒是 通过线程中断, 则直接抛出异常
                throw new InterruptedException();
            }
        }
    }finally {
        if (failed){                // 8. 在整个获取中出错(比如线程中断/超时)
            cancelAcquire(node);    // 10. 清除 node 节点(清除的过程是先给 node 打上 CANCELLED标志, 然后再删除)
        }
    }
}
```

### 7.7 释放共享lock 
当 Sync Queue中存在连续多个获取 共享lock的节点时, 会出现并发的唤醒后继节点(因为共享模式下获取lock后会唤醒近邻的后继节点来获取lock)。首先调用子类的 tryReleaseShared来进行释放 lock，然后判断是否需要唤醒后继节点来获取 lock


```java
private void doReleaseShared(){
    for(;;){
        Node h = head;                      // 1. 获取 head 节点, 准备 release
        if(h != null && h != tail){        // 2. Sync Queue 里面不为 空
            int ws = h.waitStatus;
            if(ws == Node.SIGNAL){         // 3. h节点后面可能是 独占的节点, 也可能是 共享的, 并且请求了唤醒(就是给前继节点打标记 SIGNAL)
                if(!compareAndSetWaitStatus(h, Node.SIGNAL, 0)){ // 4. h 恢复  waitStatus 值置0 (为啥这里要用 CAS 呢, 因为这里的调用可能是在 节点刚刚获取 lock, 而其他线程又对其进行中断, 所用cas就出现失败)
                    continue; // loop to recheck cases
                }
                unparkSuccessor(h);         // 5. 唤醒后继节点
            }
            else if(ws == 0 &&
                    !compareAndSetWaitStatus(h, 0, Node.PROPAGATE)){ //6. h后面没有节点需要唤醒, 则标识为 PROPAGATE 表示需要继续传递唤醒(主要是区别 独占节点最终状态0 (独占的节点在没有后继节点, 并且release lock 时最终 waitStatus 保存为 0))
                continue; // loop on failed CAS // 7. 同样这里可能存在竞争
            }
        }

        if(h == head){ // 8. head 节点没变化, 直接 return(从这里也看出, 一个共享模式的 节点在其唤醒后继节点时, 只唤醒一个, 但是它会在获取 lock 时唤醒, 释放 lock 时也进行, 所以或导致竞争的操作)
            break;           // head 变化了, 说明其他节点获取 lock 了, 自己的任务完成, 直接退出
        }

    }
}
```
## 8. 总结
本文主要讲过了抽象的队列式的同步器AQS的主要方法和实现原理。分别介绍了Node、Condition Queue、 Sync Queue、独占获取释放lock、共享获取释放lock的具体源码实现。AQS定义了一套多线程访问共享资源的同步器框架，许多同步类实现都依赖于它。 

### 参考
1. [Java并发之AQS详解](https://www.cnblogs.com/waterystone/p/4920797.html)
2. [AbstractQueuedSynchronizer 源码分析 (基于Java 8)](https://www.jianshu.com/p/e7659436538b)

