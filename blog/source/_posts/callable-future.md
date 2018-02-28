---
title: Java异步编程接口：Callable和Future
date: 2017-5-20 
categories: 并发编程
tags:
- java
---
本文主要讲解平时开发中常用的异步编程的接口：Callable和Future。

创建线程的2种方式，一种是直接继承Thread，另外一种就是实现Runnable接口。这2种方式都有一个缺陷就是：在执行完任务之后无法获取执行结果。如果需要获取执行结果，就必须通过共享变量或者使用线程通信的方式来达到效果，这样使用起来就比较麻烦。
自从Java 1.5开始，就提供了`Callable`和`Future`，通过它们可以在任务执行完毕之后得到任务执行结果。

## Callable
`Callable`接口的定义如下：

```java
public interface Callable<V> {

    V call() throws Exception;
}
```
`Callable`中定义了 `call()` 方法计算结果，或者当不能执行的时候抛出异常，可以看到返回值的类型是通过传入的泛型决定。

`Callable`并不像`Runnable`那样通过Thread的start方法就能启动实现类的run方法，所以它通常利用`ExecutorService`的submit方法去启动call方法自执行任务，而`ExecutorService`的submit又返回一个Future类型的结果，因此Callable通常也与Future一起使用

```java
 ExecutorService pool = Executors.newCachedThreadPool();
     Future<String> future = pool.submit(new Callable {
           public void call(){
              //your operations
           }
    });
```
### vs Runnable
`Runnable`与`Callable`不同点：
1. `Runnable`不返回任务执行结果，`Callable`可返回任务执行结果；
2. `Callable`在任务无法计算结果时抛出异常，而`Runnable`不能；
3. Callable支持泛型，Runnable不支持；
4. `Runnable`任务可直接由Thread的start方法或`ExecutorService`的submit方法去执行。
 
## Future
Future保存异步计算的结果,可以在我们执行任务时去做其他工作，并提供了以下几个方法：

```java
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);

    boolean isCancelled();

    boolean isDone();

    V get() throws InterruptedException, ExecutionException;

    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

* cancel(boolean mayInterruptIfRunning)：试图取消执行的任务，参数为true时直接中断正在执行的任务，否则直到当前任务执行完成，成功取消后返回true，否则返回false
* isCancel()：判断任务是否在正常执行完前被取消的，如果是则返回true
* isDone()：判断任务是否已完成
* get()：等待计算结果的返回，如果计算被取消了则抛出
* get(long timeout,TimeUtil unit)：设定计算结果的返回时间，如果在规定时间内没有返回计算结果则抛出TimeOutException

使用Future的好处：

* 获取任务的结果，判断任务是否完成，中断任务
* Future的get方法很好的替代的了Thread.join或Thread,join(long millis)
* Future的get方法可以判断程序代码(任务)的执行是否超时

### FutureTask
`FutureTask`实现了`RunnableFuture<V>`接口，关于该接口看一下其定义：

```java
public interface RunnableFuture<V> extends Runnable, Future<V> {

    void run();
}
```
`RunnableFuture`同时继承了`Runnable`和`Future`，接口中定义的`run`方法，将计算的结果设置到Future除非计算被取消。所以它既可以作为Runnable被线程执行，又可以作为Future得到Callable的返回值。
                                                                              

`FutureTask`可以直接提交给`Executor`执行，当然也可以调用线程直接执行`FutureTask.run()`。

`FutureTask`是一个可取消的异步计算，`FutureTask` 实现了Future的基本方法，提供start和cancel 操作，可以查询计算是否已经完成，并且可以获取计算的结果。
结果只可以在计算完成之后获取，get方法会阻塞当计算没有完成的时候，一旦计算已经完成， 那么计算就不能再次启动或是取消。

## 使用示例
示例场景：有一个耗时的操作，操作完后会返回一个结果（不管是正常结果还是异常），程序如果想拥有比较好的性能不可能由线程去等待操作的完成，而是应该采用listener模式。jdk并发包里的Future代表了未来的某个结果，当我们向线程池中提交任务的时候会返回该对象。
提交一个`Callable`对象给线程池时，将得到一个Future对象，并且它和传入的Callable有相同的结果类型声明。

```java
public class FutureTest {
    public static void main(String[] args) throws Throwable, ExecutionException {
        ExecutorService executor = Executors.newFixedThreadPool(2);
        //Java8 的lambada表达式，参数为Callable<T> task
        Future<String> f = executor.submit(() -> {
            System.out.println("task started!");
            Thread.sleep(1000);
            return "worker task finished!";
        });

        System.out.println("Future is ready: " + f.isDone());
        //此处阻塞main线程
        System.out.println(f.get());
        executor.shutdown();
        System.out.println("main thread is finished！");
    }
    
    //FutureTask的写法
    public static void test() throws Throwable, ExecutionException {
        FutureTask<String> f = new FutureTask<>(() -> {
            System.out.println("task started!");
            Thread.sleep(1000);
            return "worker task finished!";
        });

        Thread thread = new Thread(f);
        thread.start();

        System.out.println("Future is ready: " + f.isDone());
        //此处阻塞main线程
        System.out.println(f.get());
        System.out.println("main thread is finished！");
    }

}
```
运行结果如下：

```
Future is ready: false
task started!
worker task finished!
main thread is finished！
```
如果想获得耗时操作的结果，可以通过get方法获取，但是该方法会阻塞当前线程，我们可以在做完剩下的某些工作的时候调用get方法试图去获取结果，也可以调用非阻塞的方法isDone来确定操作是否完成。过程如下：

![](http://ovcjgn2x0.bkt.clouddn.com/callable.png )

 Future代表了线程执行完以后的结果，可以通过future获得执行的结果。但是jdk1.8之前的Future不支持，并不能实现真正的异步，需要阻塞的获取结果，或者不断的轮询。通常我们希望当线程执行完一些耗时的任务后，能够自动的通知我们结果，很遗憾这在原生jdk1.8之前
 是不支持的，但是我们可以通过第三方的库实现真正的异步回调。如Guava何Netty中都有提供，读者可以自己进行扩展，下面我们看一下JDK8中的实现。
 
 ### Java8 CompletableFuture
 虽然Future以及相关使用方法提供了异步执行任务的能力，但是对于结果的获取却是很不方便，只能通过阻塞或者轮询的方式得到任务的结果。阻塞的方式显然和我们的异步编程的初衷相违背，轮询的方式又会耗费无谓的CPU资源，而且也不能及时地得到计算结果。
 CompletableFuture类实现了CompletionStage和Future接口，所以你还是可以像以前一样通过阻塞或者轮询的方式获得结果，尽管这种方式不推荐使用。这里使用CompletableFuture实现异步的操作.
 
 ```java
public class Java8PromiseTest {
    public static void main(String[] args) throws Throwable, ExecutionException {
        ExecutorService executor = Executors.newFixedThreadPool(2);
        //jdk1.8，通过调用给定的Supplier，异步完成executor中的task
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {

            System.out.println("task started!");
            try {
                //模拟耗时操作
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "worker task is finished!";
        }, executor);

        //采用lambada的实现方式
        future.thenAccept(e -> {
            System.out.printf("%s ok", e);
            executor.shutdown();
        });

        System.out.println("main thread is finished！");
    }
}
```
运行结果如下：

```
task started!
main thread is finished！
worker task is finished! ok
```
`supplyAsync`方法以Supplier<U>函数式接口类型为参数,CompletableFuture的计算结果类型为U。`thenAccept`方法是CompletableFuture提供的一种处理结果的方法，只对结果执行Action,而不返回新的计算值，因此计算值为Void。

## 总结
本文主要介绍了异步编程经常使用的Callable、Future以及FutureTask。运行Callable任务可以拿到一个Future对象，Future 表示异步计算的结果。它提供了检查计算是否完成的方法，以等待计算的完成，并获取计算的结果。计算完成后只能使用 get 方法来获取结果，如果线程没有执行完，Future.get()方法可能会阻塞当前线程的执行；如果线程出现异常，`Future.get()`会抛出异常。
取消由cancel 方法来执行。isDone确定任务是正常完成还是被取消了。一旦计算完成，就不能再取消计算。如果为了可取消性而使用 Future 但又不提供可用的结果，则可以声明`Future<?>` 形式类型、并返回 null 作为底层任务的结果。

### 参考
1. [java异步编程](http://blog.csdn.net/tangyongzhe/article/details/49851769)
2. [Java并发编程：Callable、Future和FutureTask](http://www.importnew.com/17572.html)
