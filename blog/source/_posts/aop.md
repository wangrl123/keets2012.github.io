---
title: 深入理解Spring AOP的动态代理
date: 2017-12-14 
categories: java
tags:
- AOP
- java
---
## 1. Spring AOP
Spring是一个轻型容器，Spring整个系列的最最核心的概念当属IoC、AOP。可见AOP是Spring框架中的核心之一，在应用中具有非常重要的作用，也是Spring其他组件的基础。AOP（Aspect Oriented Programming），即面向切面编程，可以说是OOP（Object Oriented Programming，面向对象编程）的补充和完善。OOP引入封装、继承、多态等概念来建立一种对象层次结构，用于模拟公共行为的一个集合。不过OOP允许开发者定义纵向的关系，但并不适合定义横向的关系，例如日志功能。

关于AOP的基础知识，并不是本文的重点，我们主要来看下AOP的核心功能的底层实现机制：动态代理的实现原理。AOP的拦截功能是由java中的动态代理来实现的。在目标类的基础上增加切面逻辑，生成增强的目标类（该切面逻辑或者在目标类函数执行之前，或者目标类函数执行之后，或者在目标类函数抛出异常时候执行。不同的切入时机对应不同的Interceptor的种类，如BeforeAdviseInterceptor，AfterAdviseInterceptor以及ThrowsAdviseInterceptor等）。

那么动态代理是如何实现将切面逻辑（advise）织入到目标类方法中去的呢？下面我们就来详细介绍并实现AOP中用到的两种动态代理。

AOP的源码中用到了两种动态代理来实现拦截切入功能：jdk动态代理和cglib动态代理。两种方法同时存在，各有优劣。jdk动态代理是由java内部的反射机制来实现的，cglib动态代理底层则是借助asm来实现的。总的来说，反射机制在生成类的过程中比较高效，而asm在生成类之后的相关执行过程中比较高效（可以通过将asm生成的类进行缓存，这样解决asm生成类过程低效问题）。

下面我们分别来示例实现这两种方法。

## 2. JDK动态代理
### 2.1 定义接口与实现类
```java
public interface OrderService {
    public void save(UUID orderId, String name);

    public void update(UUID orderId, String name);

    public String getByName(String name);
}
```
上面代码定义了一个被拦截对象接口，即横切关注点。下面代码实现被拦截对象接口。

```java
public class OrderServiceImpl implements OrderService {

    private String user = null;

    public OrderServiceImpl() {
    }

    public OrderServiceImpl(String user) {
        this.setUser(user);
    }

	//...
	
    @Override
    public void save(UUID orderId, String name) {
        System.out.println("call save()方法,save:" + name);
    }

    @Override
    public void update(UUID orderId, String name) {
        System.out.println("call update()方法");
    }

    @Override
    public String getByName(String name) {
        System.out.println("call getByName()方法");
        return "aoho";
    }
}
```
### 2.2 JDK动态代理类 

```java
public class JDKProxy implements InvocationHandler {
	//需要代理的目标对象
    private Object targetObject;
    
    public Object createProxyInstance(Object targetObject) {
        this.targetObject = targetObject;
        return Proxy.newProxyInstance(this.targetObject.getClass().getClassLoader(),
                this.targetObject.getClass().getInterfaces(), this);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    	//被代理对象
        OrderServiceImpl bean = (OrderServiceImpl) this.targetObject;
        Object result = null;
        //切面逻辑（advise），此处是在目标类代码执行之前
        System.out.println("---before invoke----");
        if (bean.getUser() != null) {
            result = method.invoke(targetObject, args);
        }
        System.out.println("---after invoke----");
        return result;
    }

	//...

}
```

上述代码实现了动态代理类JDKProxy，实现InvocationHandler接口，并且实现接口中的invoke方法。当客户端调用代理对象的业务方法时，代理对象执行invoke方法，invoke方法把调用委派给targetObject，相当于调用目标对象的方法，在invoke方法委派前判断权限，实现方法的拦截。

### 2.3 测试
```java
public class AOPTest {
    public static void main(String[] args) {
        JDKProxy factory = new JDKProxy();
        //Proxy为InvocationHandler实现类动态创建一个符合某一接口的代理实例  
        OrderService orderService = (OrderService) factory.createProxyInstance(new OrderServiceImpl("aoho"));
		//由动态生成的代理对象来orderService 代理执行程序
        orderService.save(UUID.randomUUID(), "aoho");
    }

}
```
结果如下：

```
---before invoke----
call save()方法,save:aoho
---after invoke----
```

## 3. CGLIB字节码生成

### 3.1 要代理的类
CGLIB既可以对接口的类生成代理，也可以针对类生成代理。示例中，实现对类的代理。

```java
public class OrderManager {
    private String user = null;

    public OrderManager() {
    }

    public OrderManager(String user) {
        this.setUser(user);
    }

	//...

    public void save(UUID orderId, String name) {
        System.out.println("call save()方法,save:" + name);
    }

    public void update(UUID orderId, String name) {
        System.out.println("call update()方法");
    }

    public String getByName(String name) {
        System.out.println("call getByName()方法");
        return "aoho";
    }
}
```
该类的实现和上面的接口实现一样，为了保持统一。

### 3.2 CGLIB动态代理类

```java
public class CGLibProxy implements MethodInterceptor {
    	// CGLib需要代理的目标对象
    	private Object targetObject;

       public Object createProxyObject(Object obj) {
        this.targetObject = obj;
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(obj.getClass());
        //回调方法的参数为代理类对象CglibProxy，最后增强目标类调用的是代理类对象CglibProxy中的intercept方法 
        enhancer.setCallback(this);
        //增强后的目标类
        Object proxyObj = enhancer.create();
        // 返回代理对象
        return proxyObj;
    }

    @Override
    public Object intercept(Object proxy, Method method, Object[] args,
                            MethodProxy methodProxy) throws Throwable {
        Object obj = null;
        //切面逻辑（advise），此处是在目标类代码执行之前
        System.out.println("---before intercept----");
        obj = method.invoke(targetObject, args);
        System.out.println("---after intercept----");
        return obj;
    }
}
```
上述实现了创建子类的方法与代理的方法。getProxy(SuperClass.class)方法通过入参即父类的字节码，扩展父类的class来创建代理对象。intercept()方法拦截所有目标类方法的调用，obj表示目标类的实例，method为目标类方法的反射对象，args为方法的动态入参，methodProxy为代理类实例。method.invoke(targetObject, args)通过代理类调用父类中的方法。


### 3.3 测试

```java
public class AOPTest {
    public static void main(String[] args) {
        OrderManager order = (OrderManager) new CGLibProxy().createProxyObject(new OrderManager("aoho"));
        order.save(UUID.randomUUID(), "aoho");
    }
```
结果如下：

```
---before intercept----
call save()方法,save:aoho
---after intercept----
```

## 4. 总结

本文主要讲了Spring Aop动态代理实现的两种方式，并分别介绍了其优缺点。jdk动态代理的应用前提是目标类基于统一的接口。如果没有该前提，jdk动态代理不能应用。由此可以看出，jdk动态代理有一定的局限性，cglib这种第三方类库实现的动态代理应用更加广泛，且在效率上更有优势。

JDK动态代理机制是委托机制，不需要以来第三方的库，只要要JDK环境就可以进行代理，动态实现接口类，在动态生成的实现类里面委托为hanlder去调用原始实现类方法；CGLib 必须依赖于CGLib的类库，使用的是继承机制，是被代理类和代理类继承的关系，所以代理类是可以赋值给被代理类的，如果被代理类有接口，那么代理类也可以赋值给接口。


### 参考
1. [jdk动态代理代理与cglib代理原理探究](http://ifeve.com/jdk%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86%E4%BB%A3%E7%90%86%E4%B8%8Ecglib%E4%BB%A3%E7%90%86%E5%8E%9F%E7%90%86%E6%8E%A2%E7%A9%B6/)
2. [AOP的底层实现-CGLIB动态代理和JDK动态代理](http://blog.csdn.net/dreamrealised/article/details/12885739)



