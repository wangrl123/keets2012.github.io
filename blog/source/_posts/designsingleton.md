---
title: 设计模式之单例模式
date: 2017-02-19
categories: 设计模式
tags:
- 设计模式
---

上一篇写了[23种设计模式总览](http://blueskykong.com/2017/01/12/designPattern1/)，本文主要介绍创建模式中的单例模式，日常工作中也会有经常用到。

## 1. 定义
首先，什么是单例模式？单例模式有以下特点：

- 从字面就可以理解，单例类只能有一个实例。
- 单例类必须自己创建自己的唯一实例。
- 单例类必须给所有其他对象提供这一实例。

单例模式确保某个类只有一个实例，而且自行实例化并向整个系统提供这个实例。适用场合一般是需要频繁地进行创建和销毁的对象。如应用程序中的数据库连接池、线程池等。系统内存中该类只存在一个对象，节省了系统资源，对于一些需要频繁创建销毁的对象，使用单例模式可以提高系统性能。由于单例模式在内存中只有一个实例，减少了内存开销。 

单例模式的写法有好几种，这里主要介绍三种：懒汉式单例、饿汉式单例、登记式单例。

## 2. 实现方法
下面以java实现为例，展示几种单例模式的具体实现。

### 2.1 懒汉式单例

```java
//懒汉式单例类.在第一次调用的时候实例化自己   
public class Singleton {  
    private Singleton() {}  
    private static Singleton single=null;  
    //静态工厂方法   
    public static Singleton getInstance() {  
         if (single == null) {    
             single = new Singleton();  
         }    
        return single;  
    } 
    //... 
} 
```
上述Singleton代码通过将构造方法限定为private避免了类在外部被实例化，在同一个虚拟机范围内，Singleton的唯一实例只能通过getInstance()方法访问。（事实上，通过Java反射机制是能够实例化构造方法为private的类的，那基本上会使所有的Java单例实现失效。此问题在此处不做讨论，姑且掩耳盗铃地认为反射机制不存在。）

但是以上实现没有考虑线程安全问题。所谓线程安全是指：如果你的代码所在的进程中有多个线程在同时运行，而这些线程可能会同时运行这段代码。如果每次运行结果和单线程运行的结果是一样的，而且其他的变量的值也和预期的是一样的，就是线程安全的。或者说：一个类或者程序所提供的接口对于线程来说是原子操作或者多个线程之间的切换不会导致该接口的执行结果存在二义性,也就是说我们不用考虑同步的问题。显然以上实现并不满足线程安全的要求，在并发环境下很可能出现多个Singleton实例。

上述的实现的方式，如果现在存在着线程A和B，线程A执行到了`If(singleton == null);`，线程B执行到了`Singleton = new Singleton();`线程B虽然实例化了一个Singleton，但是对于线程A来说判断singleton还是木有初始化的，所以线程A还会对singleton进行初始化。

（1）在getInstance方法上加同步

```java
public static synchronized Singleton getInstance() {  
         if (single == null) {    
             single = new Singleton();  
         }    
        return single;  
}  
```

当线程B访问这个函数的时候，其他的任何要访问该函数的代码不能执行，直到线程B执行完该函数（这是利用锁实现的）。

(2) 双重检查锁定DCL   

synchronized似乎已经解决了多线程下的问题，但多个线程访问同一个函数的时候，那么只能有一个线程能够访问这个函数，效率很低。

```java
public static Singleton getInstance() {  
        if (singleton == null) {    
            synchronized (Singleton.class) {    
               if (singleton == null) {    
                  singleton = new Singleton();   
               }    
            }    
        }    
        return singleton;   
    }  
```

这种方式将在方法上的声明转移到了内部的代码块中，只有当singleton=null时，才需要锁机制，但是如果线程A和B同时执行到了Synchronized(singleton.class)，虽然也是只有一个线程能够执行，假如线程B先执行，线程B获得锁，线程B执行完之后，线程A获得锁，此时检查singleton是否为空再执行，所以不会出现两个singleton实例的情况。

(3) 静态内部类

关于内部类：
> 内部类都是在第一次使用时才会被加载。外部类不调用 getInstance（）时候 内部类是不会加载的，所以达到了懒汉的效果。然后调用的时候 内部类被加载，加载的时候就会初始化实例；这个加载的过程是不会有多线程的问题的！类加载的时候有一种机制叫做，缓存机制；第一次加载成功之后会被缓存起来；而且一般一个类不会加载多次。

```java
public class Singleton {    
    private static class LazyHolder {    
       private static final Singleton INSTANCE = new Singleton();    
    }    
    private Singleton (){}    
    public static final Singleton getInstance() {    
       return LazyHolder.INSTANCE;    
    }    
} 
```
 DCL优点是资源利用率高，第一次执行getInstance时单例对象才被实例化，效率高。缺点是第一次加载时反应稍慢一些，在高并发环境下也有一定的缺陷。静态内部类实现，第一次加载Singleton类时并不会初始化sInstance，只有第一次调用getInstance方法时虚拟机加载SingletonHolder 并初始化sInstance ，这样不仅能确保线程安全也能保证Singleton类的唯一性，所以推荐使用静态内部类单例模式。

但是在反序列化时会重新创建对象，将一个单例实例对象写到磁盘再读回来，从而获得了一个实例。反序列化操作提供了readResolve方法，这个方法可以让开发人员控制对象的反序列化。在上述的几个方法示例中如果要杜绝单例对象被反序列化是重新生成对象，就必须加入如下方法：

```java
private Object readResolve() throws ObjectStreamException{
	return singleton;
}
```

### 2.2 饿汉式单例

```java
public class Singleton {  
     private static Singleton instance = new Singleton();  
     private Singleton (){
     }
     public static Singleton getInstance() {  
     	return instance;  
     }  
}  
```
在类加载时就完成了初始化，所以类加载较慢，但获取对象的速度快。 这种方式基于类加载机制避免了多线程的同步问题，但是也不能确定有其他的方式（或者其他的静态方法）导致类装载，这时候初始化instance显然没有达到懒加载的效果。


### 2.3 登记式单例

```java
 //登记式单例类，利用容器实现
 //类似Spring里面的方法，将类名注册，下次从里面直接获取。
 public class SingletonManager { 
　　private static Map<String, Object> objMap = new HashMap<String,Object>();
　　private Singleton() { 
　　}
　　public static void registerService(String key, Objectinstance) {
　　　　if (!objMap.containsKey(key) ) {
　　　　　　objMap.put(key, instance) ;
　　　　}
　　}
　　public static ObjectgetService(String key) {
　　　　return objMap.get(key) ;
　　}
}
```

SingletonManager将多种的单例类统一管理，在使用时根据key获取对象对应类型的对象。这种方式使得我们可以管理多种类型的单例，并且在使用时可以通过统一的接口进行获取操作，降低了用户的使用成本，也对用户隐藏了具体实现，降低了耦合度。这种不常用，内部实现还是用的饿汉式单例，因为其中的static方法块，它的单例在类被装载的时候就被实例化了。
 
 
## 3. 总结
本文主要讲了单例模式的三种方式，分别是懒汉单例、饿汉单例和登记式单例。

主要的使用场景：

- 需要频繁的进行创建和销毁的对象；
- 创建对象时耗时过多或耗费资源过多，但又经常用到的对象；
- 工具类对象；
- 频繁访问数据库或文件的对象。

饿汉单例，类一旦加载，就把单例初始化完成，保证getInstance的时候，单例是已经存在的了。饿汉式天生就是线程安全的，可以直接用于多线程而不会出现问题。 饿汉式在类创建的同时就实例化一个静态对象出来，占据一定的内存，但是在第一次调用时速度也会更快，因为其资源已经初始化完成。

懒汉单例，只有当调用getInstance的时候，才会去初始化这个单例。是线程不安全的，在饿汉单例实现的基础上，有三种方法对多线程安全进行了处理。懒汉式会延迟加载，在第一次使用该单例的时候才会实例化对象出来，第一次调用时要做初始化，如果要做的工作比较多，性能上会有些延迟，之后就和饿汉式一样了。


---
### 参考
1. [JAVA设计模式之单例模式](http://blog.csdn.net/jason0539/article/details/23297037/#reply)
2. [高并发下线程安全的单例模式（最全最经典）](http://blog.csdn.net/cselmu9/article/details/51366946)


