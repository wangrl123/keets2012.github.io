---
title: 基于redis的分布式锁实现
date: 2018-1-6 
categories: 并发编程
tags:
- java
- redis
---
## 关于分布式锁
很久之前有讲过并发编程中的锁[并发编程的锁机制：synchronized和lock](http://blueskykong.com/2017/05/14/lock/)。在单进程的系统中，当存在多个线程可以同时改变某个变量时，就需要对变量或代码块做同步，使其在修改这种变量时能够线性执行消除并发修改变量。而同步的本质是通过锁来实现的。为了实现多个线程在一个时刻同一个代码块只能有一个线程可执行，那么需要在某个地方做个标记，这个标记必须每个线程都能看到，当标记不存在时可以设置该标记，其余后续线程发现已经有标记了则等待拥有标记的线程结束同步代码块取消标记后再去尝试设置标记。

分布式环境下，数据一致性问题一直是一个比较重要的话题，而又不同于单进程的情况。分布式与单机情况下最大的不同在于其不是多线程而是多进程。多线程由于可以共享堆内存，因此可以简单的采取内存作为标记存储位置。而进程之间甚至可能都不在同一台物理机上，因此需要将标记存储在一个所有进程都能看到的地方。

常见的是秒杀场景，订单服务部署了多个实例。如秒杀商品有4个，第一个用户购买3个，第二个用户购买2个，理想状态下第一个用户能购买成功，第二个用户提示购买失败，反之亦可。而实际可能出现的情况是，两个用户都得到库存为4，第一个用户买到了3个，更新库存之前，第二个用户下了2个商品的订单，更新库存为2，导致出错。

在上面的场景中，商品的库存是共享变量，面对高并发情形，需要保证对资源的访问互斥。在单机环境中，Java中其实提供了很多并发处理相关的API，但是这些API在分布式场景中就无能为力了。也就是说单纯的Java Api并不能提供分布式锁的能力。分布式系统中，由于分布式系统的分布性，即多线程和多进程并且分布在不同机器中，synchronized和lock这两种锁将失去原有锁的效果，需要我们自己实现分布式锁。

常见的锁方案如下：

- 基于数据库实现分布式锁 
- 基于缓存，实现分布式锁，如redis
- 基于Zookeeper实现分布式锁

下面我们简单介绍下这几种锁的实现。

### 基于数据库
基于数据库的锁实现也有两种方式，一是基于数据库表，另一种是基于数据库排他锁。

#### 基于数据库表的增删
基于数据库表增删是最简单的方式，首先创建一张锁的表主要包含下列字段：方法名，时间戳等字段。

具体使用的方法，当需要锁住某个方法时，往该表中插入一条相关的记录。这边需要注意，方法名是有唯一性约束的，如果有多个请求同时提交到数据库的话，数据库会保证只有一个操作可以成功，那么我们就可以认为操作成功的那个线程获得了该方法的锁，可以执行方法体内容。

执行完毕，需要delete该记录。

当然，笔者这边只是简单介绍一下。对于上述方案可以进行优化，如应用主从数据库，数据之间双向同步。一旦挂掉快速切换到备库上；做一个定时任务，每隔一定时间把数据库中的超时数据清理一遍；使用while循环，直到insert成功再返回成功，虽然并不推荐这样做；还可以记录当前获得锁的机器的主机信息和线程信息，那么下次再获取锁的时候先查询数据库，如果当前机器的主机信息和线程信息在数据库可以查到的话，直接把锁分配给他就可以了，实现可重入锁。

#### 基于数据库排他锁
我们还可以通过数据库的排他锁来实现分布式锁。基于MySql的InnoDB引擎，可以使用以下方法来实现加锁操作：

```java
public void lock(){
    connection.setAutoCommit(false)
    int count = 0;
    while(count < 4){
        try{
            select * from lock where lock_name=xxx for update;
            if(结果不为空){
                //代表获取到锁
                return;
            }
        }catch(Exception e){

        }
        //为空或者抛异常的话都表示没有获取到锁
        sleep(1000);
        count++;
    }
    throw new LockException();
}
```

在查询语句后面增加for update，数据库会在查询过程中给数据库表增加排他锁。当某条记录被加上排他锁之后，其他线程无法再在该行记录上增加排他锁。其他没有获取到锁的就会阻塞在上述select语句上，可能的结果有2种，在超时之前获取到了锁，在超时之前仍未获取到锁。

获得排它锁的线程即可获得分布式锁，当获取到锁之后，可以执行方法的业务逻辑，执行完方法之后，释放锁`connection.commit()`。

存在的问题主要是性能不高和sql超时的异常。

#### 基于数据库锁的优缺点 
上面两种方式都是依赖数据库的一张表，一种是通过表中的记录的存在情况确定当前是否有锁存在，另外一种是通过数据库的排他锁来实现分布式锁。

- 优点是直接借助数据库，简单容易理解。
- 缺点是操作数据库需要一定的开销，性能问题需要考虑。

### 基于Zookeeper
基于zookeeper临时有序节点可以实现的分布式锁。每个客户端对某个方法加锁时，在zookeeper上的与该方法对应的指定节点的目录下，生成一个唯一的瞬时有序节点。 判断是否获取锁的方式很简单，只需要判断有序节点中序号最小的一个。 当释放锁的时候，只需将这个瞬时节点删除即可。同时，其可以避免服务宕机导致的锁无法释放，而产生的死锁问题。

提供的第三方库有[curator](https://curator.apache.org/)，具体使用读者可以自行去看一下。Curator提供的InterProcessMutex是分布式锁的实现。acquire方法获取锁，release方法释放锁。另外，锁释放、阻塞锁、可重入锁等问题都可以有有效解决。讲下阻塞锁的实现，客户端可以通过在ZK中创建顺序节点，并且在节点上绑定监听器，一旦节点有变化，Zookeeper会通知客户端，客户端可以检查自己创建的节点是不是当前所有节点中序号最小的，如果是就获取到锁，便可以执行业务逻辑。

最后，Zookeeper实现的分布式锁其实存在一个缺点，那就是性能上可能并没有缓存服务那么高。因为每次在创建锁和释放锁的过程中，都要动态创建、销毁瞬时节点来实现锁功能。ZK中创建和删除节点只能通过Leader服务器来执行，然后将数据同不到所有的Follower机器上。并发问题，可能存在网络抖动，客户端和ZK集群的session连接断了，zk集群以为客户端挂了，就会删除临时节点，这时候其他客户端就可以获取到分布式锁了。

### 基于缓存
相对于基于数据库实现分布式锁的方案来说，基于缓存来实现在性能方面会表现的更好一点，存取速度快很多。而且很多缓存是可以集群部署的，可以解决单点问题。基于缓存的锁有好几种，如memcached、redis、本文下面主要讲解基于redis的分布式实现。

## 基于redis的分布式锁实现
### SETNX
使用redis的SETNX实现分布式锁，多个进程执行以下Redis命令：

```
SETNX lock.id <current Unix time + lock timeout + 1>
```
SETNX是将 key 的值设为 value，当且仅当 key 不存在。若给定的 key 已经存在，则 SETNX 不做任何动作。   

- 返回1，说明该进程获得锁，SETNX将键 lock.id 的值设置为锁的超时时间，当前时间 +加上锁的有效时间。 
- 返回0，说明其他进程已经获得了锁，进程不能进入临界区。进程可以在一个循环中不断地尝试 SETNX 操作，以获得锁。

### 存在死锁的问题
SETNX实现分布式锁，可能会存在死锁的情况。与单机模式下的锁相比，分布式环境下不仅需要保证进程可见，还需要考虑进程与锁之间的网络问题。某个线程获取了锁之后，断开了与Redis 的连接，锁没有及时释放，竞争该锁的其他线程都会hung，产生死锁的情况。

在使用 SETNX 获得锁时，我们将键 lock.id 的值设置为锁的有效时间，线程获得锁后，其他线程还会不断的检测锁是否已超时，如果超时，等待的线程也将有机会获得锁。然而，锁超时，我们不能简单地使用 DEL 命令删除键 lock.id 以释放锁。

考虑以下情况:
> 1. A已经首先获得了锁 lock.id，然后线A断线。B,C都在等待竞争该锁；
> 2. B,C读取lock.id的值，比较当前时间和键 lock.id 的值来判断是否超时，发现超时；
> 3. B执行 DEL lock.id命令，并执行 SETNX lock.id 命令，并返回1，B获得锁；
> 4. C由于各刚刚检测到锁已超时，执行 DEL lock.id命令，将B刚刚设置的键 lock.id 删除，执行 SETNX lock.id命令，并返回1，即C获得锁。

上面的步骤很明显出现了问题，导致B,C同时获取了锁。在检测到锁超时后，线程不能直接简单地执行 DEL 删除键的操作以获得锁。

对于上面的步骤进行改进，问题是出在删除键的操作上面，那么获取锁之后应该怎么改进呢？
首先看一下redis的GETSET这个操作，`GETSET key value`，将给定 key 的值设为 value ，并返回 key 的旧值(old value)。利用这个操作指令，我们改进一下上述的步骤。

> 1. A已经首先获得了锁 lock.id，然后线A断线。B,C都在等待竞争该锁；
> 2. B,C读取lock.id的值，比较当前时间和键 lock.id 的值来判断是否超时，发现超时；
> 3. B检测到锁已超时，即当前的时间大于键 lock.id 的值，B会执行
`GETSET lock.id <current Unix timestamp + lock timeout + 1>`设置时间戳，通过比较键 lock.id 的旧值是否小于当前时间，判断进程是否已获得锁；
> 4. B发现GETSET返回的值小于当前时间，则执行 DEL lock.id命令，并执行 SETNX lock.id 命令，并返回1，B获得锁；
> 5. C执行GETSET得到的时间大于当前时间，则继续等待。

在线程释放锁，即执行 DEL lock.id 操作前，需要先判断锁是否已超时。如果锁已超时，那么锁可能已由其他线程获得，这时直接执行 DEL lock.id 操作会导致把其他线程已获得的锁释放掉。

### 一种实现方式

#### 获取锁

```java
public boolean lock(long acquireTimeout, TimeUnit timeUnit) throws InterruptedException {
    acquireTimeout = timeUnit.toMillis(acquireTimeout);
    long acquireTime = acquireTimeout + System.currentTimeMillis();
    //使用J.U.C的ReentrantLock
    threadLock.tryLock(acquireTimeout, timeUnit);
    try {
    	//循环尝试
        while (true) {
        	//调用tryLock
            boolean hasLock = tryLock();
            if (hasLock) {
                //获取锁成功
                return true;
            } else if (acquireTime < System.currentTimeMillis()) {
                break;
            }
            Thread.sleep(sleepTime);
        }
    } finally {
        if (threadLock.isHeldByCurrentThread()) {
            threadLock.unlock();
        }
    }

    return false;
}

public boolean tryLock() {

    long currentTime = System.currentTimeMillis();
    String expires = String.valueOf(timeout + currentTime);
    //设置互斥量
    if (redisHelper.setNx(mutex, expires) > 0) {
    	//获取锁，设置超时时间
        setLockStatus(expires);
        return true;
    } else {
        String currentLockTime = redisUtil.get(mutex);
        //检查锁是否超时
        if (Objects.nonNull(currentLockTime) && Long.parseLong(currentLockTime) < currentTime) {
            //获取旧的锁时间并设置互斥量
            String oldLockTime = redisHelper.getSet(mutex, expires);
            //旧值与当前时间比较
            if (Objects.nonNull(oldLockTime) && Objects.equals(oldLockTime, currentLockTime)) {
            	//获取锁，设置超时时间
                setLockStatus(expires);
                return true;
            }
        }

        return false;
    }
}
```
lock调用tryLock方法，参数为获取的超时时间与单位，线程在超时时间内，获取锁操作将自旋在那里，直到该自旋锁的保持者释放了锁。

tryLock方法中，主要逻辑如下：

- setnx(lockkey, 当前时间+过期超时时间) ，如果返回1，则获取锁成功；如果返回0则没有获取到锁
- get(lockkey)获取值oldExpireTime ，并将这个value值与当前的系统时间进行比较，如果小于当前系统时间，则认为这个锁已经超时，可以允许别的请求重新获取
- 计算newExpireTime=当前时间+过期超时时间，然后getset(lockkey, newExpireTime) 会返回当前lockkey的值currentExpireTime
- 判断currentExpireTime与oldExpireTime 是否相等，如果相等，说明当前getset设置成功，获取到了锁。如果不相等，说明这个锁又被别的请求获取走了，那么当前请求可以直接返回失败，或者继续重试


#### 释放锁

```java
    public boolean unlock() {
        //只有锁的持有线程才能解锁
        if (lockHolder == Thread.currentThread()) {
            //判断锁是否超时，没有超时才将互斥量删除
            if (lockExpiresTime > System.currentTimeMillis()) {
                redisHelper.del(mutex);
                logger.info("删除互斥量[{}]", mutex);
            }
            lockHolder = null;
            logger.info("释放[{}]锁成功", mutex);

            return true;
        } else {
            throw new IllegalMonitorStateException("没有获取到锁的线程无法执行解锁操作");
        }
    }
```

在上面获取锁的实现下，其实此处的释放锁函数可以不需要了，有兴趣的读者可以结合上面的代码看下为什么？有想法可以留言哦！

## 总结
本文主要讲解了基于redis分布式锁的实现，在分布式环境下，数据一致性问题一直是一个比较重要的话题，而synchronized和lock锁在分布式环境已经失去了作用。常见的锁的方案有基于数据库实现分布式锁、基于缓存实现分布式锁、基于Zookeeper实现分布式锁，简单介绍了每种锁的实现特点；然后，文中探索了一下redis锁的实现方案；最后，本文给出了基于Java实现的redis分布式锁，读者可以自行验证一下。


### 参考
1. [分布式锁的一点理解](https://www.cnblogs.com/suolu/p/6588902.html)
2. [分布式锁1 Java常用技术方案](https://www.cnblogs.com/PurpleDream/p/5559352.html)
3. [分布式锁的几种实现方式](https://www.cnblogs.com/austinspark-jessylu/p/8043726.html)
