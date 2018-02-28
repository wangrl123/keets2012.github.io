---
title: Callable和Future源码解析
date: 2017-5-26 
categories: 并发编程
tags:
- java
---
在上一篇[Java异步编程接口：Callable和Future](http://blueskykong.com/2017/05/20/callable-future/)中介绍了异步编程经常使用的Callable、Future以及FutureTask，并给出了应用示例。本文主要在上一篇的基础应用基础上，深入源码分析实现原理。本文基于JDK8。

## 异步接口
### Callable接口
许多任务实际上都是存在延迟的计算，对于这些任务，`Callable`是一种更好的抽象：它会返回一个值，并可能抛出一个异常。Callable接口：

```java
public interface Callable<V> {

    V call() throws Exception;
}
```
该接口中唯一的方法`call()`返回的类型就是传递进来的V类型。

`Runnable`和`Callable`描述的都是抽象的计算任务。这些任务通常是有生命周期的。
### Future接口

```java
public interface Future<V> {
    //取消任务，
    boolean cancel(boolean mayInterruptIfRunning);

    boolean isCancelled();

    boolean isDone();

    V get() throws InterruptedException, ExecutionException;


    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```
Executor执行的任务有4个生命周期阶段：创建、提交、开始和完成。
由于有些任务可能要执行很长时间，因此通常希望可以取消这些任务。在Executor框架中，已提交但尚未开始的任务可以取消，对于已经开始执行的任务，只有当它们响应中断时才能取消。

实际上Future提供了三种功能：判断任务是否完成、中断任务和获取任务执行结果。

## 如何实现异步
在不阻塞当前线程的情况下计算，那么必然需要另外的线程去执行具体的业务逻辑，上面代码中可以看到，是把Callable放入了线程池中，等待执行，并且立刻返回futrue。可以猜想下，需要从Future中得到Callable的结果，那么Future的引用必然会被两个线程共享，一个线程执行完成后改变Future的状态位并唤醒挂起在get上的线程，到底是不是这样呢？

`ExecutorService`接口中定义了`submit()`用以提交任务，在`AbstractExecutorService`中的源码如下：
                                        
```java
    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask);
        return ftask;
    }
    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        return new FutureTask<T>(runnable, value);
    }
    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW; 
    }
```
可以看到Callable任务被包装成了RunnableFuture对象，通过了线程池的execute方法提交任务并且立刻返回对象本身，而线程池是接受Runnable，RunnableFuture继承了Runnable。

```java
public interface RunnableFuture<V> extends Runnable, Future<V> {

    void run();
}
```
设置计算的结果到Future中，除非任务被取消。可以看到，`newTaskFor`返回的是`FutureTask`，构造函数的参数为Callable任务。我们具体看一下`FutureTask`的实现，不过可以肯定的是该类实现了`RunnableFuture`接口。

### FutureTask
`FutureTask`被生产者和消费者共享，生产者运行`run()`方法计算结果，消费者通过get方法获取结果，那么必然就需要通知，如何通知呢，肯定是状态位变化，并唤醒线程。

```java
    private volatile int state;
    //在构建FutureTask时设置，同时也表示内部成员callable已成功赋值
    private static final int NEW          = 0;
    //woker thread在处理task时设定的中间状态
    private static final int COMPLETING   = 1;
    //当设置result结果完成后，FutureTask处于该状态，代表过程结果
    private static final int NORMAL       = 2;
    //task执行过程出现异常，此时结果设值为exception,也是final state
    private static final int EXCEPTIONAL  = 3;
    //表明task被cancel
    private static final int CANCELLED    = 4;
    //中间状态，task运行过程中被interrupt时，设置的中间状态
    private static final int INTERRUPTING = 5;
    //中断完毕的最终状态
    private static final int INTERRUPTED  = 6;
```
state初始化为NEW。只有在set, setException和cancel方法中state才可以转变为终态。在任务完成期间，state的值可能为COMPLETING或INTERRUPTING。 state存在可能的四种状态变化如下：

```
NEW -> COMPLETING -> NORMAL
NEW -> COMPLETING -> EXCEPTIONAL
NEW -> CANCELLED
NEW -> INTERRUPTING -> INTERRUPTED
```

### Task的状态变化
`FutureTask`有两个构造函数，分别接受`Callable`和`Runnable`作为创建任务的参数。

```java
    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;       // ensure visibility of callable
    }
```

```java
    public FutureTask(Runnable runnable, V result) {
        //result，成功计算后返回的结果类型
        this.callable = Executors.callable(runnable, result);
        this.state = NEW;       // ensure visibility of callable
     }
```
此时将state设置为初始态NEW。这里注意`Runnable`是怎样转换为`Callable`的，看下`Executors.callable(runnable, result)`的具体实现：

```java
public static <T> Callable<T> callable(Runnable task, T result) {
        if (task == null)
            throw new NullPointerException();
        return new RunnableAdapter<T>(task, result);
}

static final class RunnableAdapter<T> implements Callable<T> {
        final Runnable task;
        final T result;
        RunnableAdapter(Runnable task, T result) {
            this.task = task;
            this.result = result;
        }
        public T call() {
            task.run();
            return result;
        }
}
```
上面实现可以看出通过Callable的call方法调用Runnable的run方法，把传入的 T result 作为callable的返回结果。

当创建完一个Task通常会提交给`Executors`来执行，当然也可以使用`Thread`来执行，`Thread`的`start()`方法会调用Task的run()方法。

```java
public void run() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
```
首先判断任务的状态，如果任务状态不是new，说明任务状态已经改变。可能是上面4种可能变化的一种，比如caller调用了cancel，此时状态为Interrupting, 也说明了上面的cancel方法，task没运行时，就interrupt, task得不到运行，总是返回。
如果状态是new, 判断runner是否为null, 如果为null, 则把当前执行任务的线程赋值给runner，如果runner不为null, 说明已经有线程在执行，返回。
此处使用cas（使用`compareAndSwap`能够保证原子性）来赋值worker thread是保证多个线程同时提交同一个`FutureTask`时，确保该FutureTask的run只被调用一次， 如果想运行多次，使用`runAndReset()`方法。

接着开始执行任务，如果要执行的任务不为空，并且state为New就执行，可以看到这里调用了Callable的call方法。如果执行成功则set结果，如果出现异常则setException,最后把runner设为null。看一下set方法的实现：

```java
protected void set(V v) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = v;
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
            finishCompletion();
        }
}
```

如果现在的状态是NEW就把状态设置成COMPLETING，然后设置成NORMAL。这个执行流程的状态变化就是： `NEW->COMPLETING->NORMAL`。最后执行`finishCompletion`方法：

```java
private void finishCompletion() {
        // assert state > COMPLETING;
        for (WaitNode q; (q = waiters) != null;) {
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }
        done();
        callable = null;        // to reduce footprint
 }
```
该方法将会解除所有阻塞的worker thread， 调用done()方法，将成员变量callable设为null。

### 等待的线程
如何挂起get()方法的线程，看一下具体实现：

```java
public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        return report(s);
}
```
首先判断FutureTask的状态是否为完成状态，如果是完成状态，说明已经执行过set或setException方法，返回report(s):

```java
private V report(int s) throws ExecutionException {
        Object x = outcome;
        if (s == NORMAL)
            return (V)x;
        if (s >= CANCELLED)
            throw new CancellationException();
        throw new ExecutionException((Throwable)x);
}
```
可以看到，如果FutureTask的状态是NORMAL, 即正确执行了set方法，get方法直接返回处理的结果， 如果是取消状态，即执行了setException，则抛出CancellationException异常。如果get时,FutureTask的状态为未完成状态，则调用awaitDone方法进行阻塞。awaitDone():
                                                                                                             
```java
private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        WaitNode q = null;
        boolean queued = false;
        for (;;) {
            if (Thread.interrupted()) {
                removeWaiter(q);
                throw new InterruptedException();
            }

            int s = state;
            if (s > COMPLETING) {
                if (q != null)
                    q.thread = null;
                return s;
            }
            else if (s == COMPLETING) 
                Thread.yield();
            else if (q == null)
                q = new WaitNode();
            else if (!queued)
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                     q.next = waiters, q);
            else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                    removeWaiter(q);
                    return state;
                }
                LockSupport.parkNanos(this, nanos);
            }
            else
                LockSupport.park(this);
        }
}
```

awaitDone方法可以看成是不断轮询查看FutureTask的状态。在get阻塞期间：

- 如果执行get的线程被中断，则移除FutureTask的所有阻塞队列中的线程（waiters）,并抛出中断异常；
- 如果FutureTask的状态转换为完成状态（正常完成或取消），则返回完成状态；
- 如果FutureTask的状态变为COMPLETING, 则说明正在set结果，此时让线程等一等；
- 如果FutureTask的状态为初始态NEW，则将当前线程加入到FutureTask的阻塞线程中去；
- 如果get方法没有设置超时时间，则阻塞当前调用get线程；如果设置了超时时间，则判断是否达到超时时间，如果到达，则移除FutureTask的所有阻塞列队中的线程，并返回此时FutureTask的状态，如果未到达时间，则在剩下的时间内继续阻塞当前线程。

## 总结
本文主要讲了Java异步接口的实现原理。从我们的猜想开始，在不阻塞当前线程的情况下计算，那么必然需要另外的线程去执行具体的业务逻辑，从具体应用时可以看到，是把Callable放入了线程池中，等待执行，并且立刻返回futrue。现在需要从Future中得到Callable的结果，那么Future的引用必然会被两个线程共享，一个线程执行完成后改变Future的状态位并唤醒挂起在get上的线程。由`ExecutorService`的提交任务，逐步分析到`FutureTask`中状态位的设置以及状态的变化，最后分析了
`get()`方法获取计算结果时，在`FutureTask`中如何挂起并最终返回计算结果。


### 参考
1. [Callable异步原理简析](http://blog.csdn.net/u012664375/article/details/66967687)
2. [Callable、Future和FutureTask原理解析](http://blog.csdn.net/codershamo/article/details/51901057)
3. [JDK8 API]()
