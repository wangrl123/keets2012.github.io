---
title: 设计模式之代理模式
date: 2017-02-22
categories: 设计模式
tags:
- 设计模式
---
代理模式属于结构性模式。
## 代理模式的定义
代理模式为其他对象提供一种代理以控制对这个对象的访问。从定义可以知道代理模式控制客户端对一个对象的访问，它跟现实中的中介代理类似，只是作为代表做一些受理工作，真正执行的并不是它自己。
## 代理模式的结构
![proxymode](http://ovcjgn2x0.bkt.clouddn.com/proxymode.jpg "代理模式")

代理模式中的角色：

- 抽象主题 Subject：声明了目标对象和代理对象的共同接口，任何可以使用目标对象的地方都可以使用代理对象。
- 具体主题 RealSubject：也称为委托角色或者被代理角色。定义了代理对象所代表的目标对象。
- 代理主题 Proxy：代理类。代理对象内部含有目标对象的引用，从而可以操作目标对象；代理对象提供一个与目标对象相同的接口，以便替代目标对象。代理对象通常在客户端调用传递给目标对象之前或之后，执行某个操作，而不是单纯地将调用传递给目标对象。 

## 代理模式的分类
代理模式又分为静态代理和动态代理。

### 静态代理
静态代理是由开发创建或特定工具自动生成源代码，再对其编译。在程序运行前，代理类的`.class`文件就已经存在了。

主题接口的定义：

```java
public interface Subject
{
    void hello();
}
```
具体主题类的定义：

```java
public class RealSubject implements Subject
{
    @Override
    public void hello()
    {
        System.out.println("RealSubject");
    }
}
```
代理主题的实现：

```java
public class Proxy implements Subject
{
    private Subject subject = null;

    @Override
    public void hello()
    {
        if(subject == null)
            subject = new RealSubject();
        System.out.print("Hello, I'm A Proxy, I'm invoking...");
        this.subject.hello();
    }
}
```
调用时只需要实例化代理对象即可`Subject subject = new Proxy();`。代理对象将客户端的调用指派给目标对象，在调用目标对象的方法之前和之后都可以执行特定的操作。

### 动态代理
Java动态代理有两种实现：jdk动态代理和cglib动态代理。
#### JDK动态代理
jdk动态代理是由java内部的反射机制来实现的。

- 定义接口与实现类

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

- JDK动态代理类 

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

- 测试

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

#### CGLIB动态代理
cglib动态代理底层则是借助asm来实现的。

- 要代理的类
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

- CGLIB动态代理类

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


- 测试

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

## 总结
本文主要介绍了创建型的模式：代理模式。在某些情况下，一个对象不适合或者不能直接引用另一个对象，而代理对象可以在客户端和目标对象之间起到中介的作用。代理模式又分为静态代理和动态代理，不同之处在于代理类的生成时间。动态代理又分为jdk动态代理和cglib动态代理实现。两种方法各有优劣。总的来说，反射机制在生成类的过程中比较高效，而asm在生成类之后的相关执行过程中比较高效（可以通过将asm生成的类进行缓存，这样解决asm生成类过程低效问题）。

### vs 装饰者模式
装饰者模式与代理模式很类似，二者最主要的区别是：代理模式中，代理类对被代理的对象有控制权，决定其执行或者不执行。而装饰模式中，装饰类对代理对象没有控制权，只能为其增加一层装饰，以加强被装饰对象的功能，仅此而已。装饰者模式主要是用来增加类的职责和行为的，将类的核心职责和装饰功能区分开，可以很方便对装饰功能进行添加和去除。

### vs 中介者模式
中介者模式用一个中介者对象来封装一系列对象的交互。中介者使得各对象不需要显式地相互引用，从而解耦合，独立改变他们之间的交互。主要区别如下：

- 代理模式是一对一，一个代理只能代表一个对象。中介者模式则是多对多，中介者的功能多样，客户也可以多个。
- 只能代理一方。如果PB是A的代理，那么C可以通过PB访问A，但是A不能通过PB访问B。对于中介者模式而言，A可以通过中介访问B，B也可以通过中介访问A。

### vs 外观模式
和上面类似，代理对象代表一个单一对象，而外观对象代表一个子系统，代理的客户对象无法直接访问对象，由代理提供单独的目标对象的访问，而通常外观对象提供对子系统各元件功能的简化的共同层次的调用接口。代理是一种原来对象的代表，其他需要与这个对象打交道的操作都是和这个代表交涉的。

### 参考
1. [设计模式：代理模式](http://blog.csdn.net/u013256816/article/details/51009592)
2. [Java设计模式之代理模式](https://www.cnblogs.com/liaoweipeng/p/5342214.html)
3. [中介者模式、代理模式和外观模式的Pk](http://blog.csdn.net/mengmei16/article/details/43981791)
