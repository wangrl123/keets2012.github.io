---
title: 深入ThreadLocal
date: 2017-9-4 
categories: Note
tags:
- Thread
- java
---
ThreadLocal主要是提供线程内部的局部变量，在`每个`线程内随时随地可取，隔离其他线程。   

## 1. ThreadLocal接口
### 1.1 ThreadLocal类接口很简单，只有4个方法，我们先来了解一下：

- `void set(Object value)`设置当前线程的线程局部变量的值。
- `public Object get()`该方法返回当前线程所对应的线程局部变量。
- `public void remove()`将当前线程局部变量的值删除，目的是为了减少内存的占用，该方法是JDK 5.0新增的方法。需要指出的是，当线程结束后，对应该线程的局部变量将自动被垃圾回收，所以显式调用该方法清除线程的局部变量并不是必须的操作，但它可以加快内存回收的速度。
- `protected Object initialValue()`返回该线程局部变量的初始值，该方法是一个protected的方法，显然是为了让子类覆盖而设计的。这个方法是一个延迟调用方法，在线程第1次调用get()或set(Object)时才执行，并且仅执行1次。ThreadLocal中的缺省实现直接返回一个null。

在同步机制中，通过对象的锁机制保证同一时间只有一个线程访问变量。这时该变量是多个线程共享的，使用同步机制要求程序慎密地分析什么时候对变量进行读写，什么时候需要锁定某个对象，什么时候释放对象锁等繁杂的问题，程序设计和编写难度相对较大。   

而ThreadLocal则从另一个角度来解决多线程的并发访问。ThreadLocal会为每一个线程提供一个独立的变量副本，从而隔离了多个线程对数据的访问冲突。因为每一个线程都拥有自己的变量副本，从而也就没有必要对该变量进行同步了。ThreadLocal提供了线程安全的共享对象，在编写多线程代码时，可以把不安全的变量封装进ThreadLocal。   
如果想在get之前不需要调用set就能正常访问的话，必须重写initialValue()方法。最常见的ThreadLocal使用场景为 用来解决 数据库连接、Session管理等。

### 1.2 使用ThreadLocal
```java
public class ThreadTest {
    ThreadLocal<Long> longLocal = new ThreadLocal<Long>(){
        protected Long initialValue() {
            return Thread.currentThread().getId();
        }
    };
    ThreadLocal<String> stringLocal = new ThreadLocal<String>(){;
        protected String initialValue() {
            return Thread.currentThread().getName();
        }
    };

    public void set() {
        longLocal.set(Thread.currentThread().getId());
        stringLocal.set(Thread.currentThread().getName());
    }

    public long getLong() {
        return longLocal.get();
    }

    public String getString() {
        return stringLocal.get();
    }

    public static void main(String[] args) throws InterruptedException {
        final ThreadTest test = new ThreadTest();


        //test.set();
        System.out.println("main.getLong: " + test.getLong());
        System.out.println("main.getString: " + test.getString());


        Thread thread1 = new Thread() {
            public void run() {
                //test.set();
                System.out.println("Thread.getLong: " + test.getLong());
                System.out.println("Thread.getString: " + test.getString());
            }
        };
        thread1.start();
        thread1.join();
    }
}
```

以上demo覆写了initialValue()方法，或者调用set方法，否则会报空指针异常。在main线程中和thread1线程中，longLocal保存的副本值和stringLocal保存的副本值都不一样。
## 2. ThreadLocalMap
ThreadLocalMap的Entry继承了WeakReference，并且使用ThreadLocal作为键值。   
　　1. 实际的通过ThreadLocal创建的副本是存储在每个线程自己的threadLocals中的；   
　　2. 为何threadLocals的类型ThreadLocalMap的键值为ThreadLocal对象，因为每个线程中可有多个threadLocal变量，就像上面代码中的longLocal和stringLocal；   
　　3. 在进行get之前，必须先set，否则会报空指针异常；
　
### 2.1 方法分析
1). JDK8的ThreadLocal的get方法的源码

```java
    /**
     * Returns the value in the current thread's copy of this
     * thread-local variable.  If the variable has no value for the
     * current thread, it is first initialized to the value returned
     * by an invocation of the {@link #initialValue} method.
     *
     * @return the current thread's value of this thread-local
     */
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```

2). getMap 
  
```java
    /**
     * Get the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *
     * @param  t the current thread
     * @return the map
     */
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```

3). setInitialValue

```java
    /**
     * Variant of set() to establish initialValue. Used instead
     * of set() in case user has overridden the set() method.
     *
     * @return the initial value
     */
    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
```
get方法的流程是这样的：

- 1.首先获取当前线程
- 2.根据当前线程获取一个Map
- 3.如果获取的Map不为空，则在Map中以ThreadLocal的引用作为key来在Map中获取对应的value e，否则转到5
- 4.如果e不为null，则返回e.value，否则转到5
- 5.Map为空或者e为空，则通过initialValue函数获取初始值value，然后用ThreadLocal的引用和value作为firstKey和firstValue创建一个新的Map

每个Thread维护一个ThreadLocalMap映射表，这个映射表的key是ThreadLocal实例本身，value是真正需要存储的Object。

## 3. WeakReference
关于内存泄露：
> ThreadLocalMap使用ThreadLocal的弱引用作为key，如果一个ThreadLocal没有外部强引用引用他，那么系统gc的时候，这个ThreadLocal势必会被回收，这样一来，ThreadLocalMap中就会出现key为null的Entry，就没有办法访问这些key为null的Entry的value，如果当前线程再迟迟不结束的话，这些key为null的Entry的value就会一直存在一条强引用链：   
> `ThreadLocal Ref -> Thread -> ThreaLocalMap -> Entry -> value`

主要看下getEntryAfterMiss函数：

```java
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;

            while (e != null) {
                ThreadLocal<?> k = e.get();
                if (k == key)
                    return e;
                if (k == null)
                    expungeStaleEntry(i);
                else
                    i = nextIndex(i, len);
                e = tab[i];
            }
            return null;
        }
```

ThreadLocalMap的getEntry函数的流程：

1. 首先从ThreadLocal的直接索引位置(通过ThreadLocal.threadLocalHashCode & (len-1)运算得到)获取Entry e，如果e不为null并且key相同则返回e；
2. 如果e为null或者key不一致则向下一个位置查询，如果下一个位置的key和当前需要查询的key相等，则返回对应的Entry，否则，如果key值为null，则擦除该位置的Entry，否则继续向下一个位置查询

**在这个过程中遇到的key为null的Entry都会被擦除，那么Entry内的value也就没有强引用链，自然会被回收。仔细研究代码可以发现，set操作也有类似的思想，将key为null的这些Entry都删除，防止内存泄露。 但是光这样还是不够的，上面的设计思路依赖一个前提条件：要调用ThreadLocalMap的genEntry函数或者set函数。这当然是不可能任何情况都成立的，所以很多情况下需要使用者手动调用ThreadLocal的remove函数，手动删除不再需要的ThreadLocal，防止内存泄露。所以JDK建议将ThreadLocal变量定义成private static的，这样的话ThreadLocal的生命周期就更长，由于一直存在ThreadLocal的强引用，所以ThreadLocal也就不会被回收，也就能保证任何时候都能根据ThreadLocal的弱引用访问到Entry的value值，然后remove它，防止内存泄露。**

---
参考：
[ThreadLocal和synchronized的区别](https://www.zhihu.com/question/23089780)   
[Java并发编程：深入剖析ThreadLocal](http://www.cnblogs.com/dolphin0520/p/3920407.html)



