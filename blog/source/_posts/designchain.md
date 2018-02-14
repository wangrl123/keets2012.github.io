---
title: 设计模式之责任链模式
date: 2017-03-25
categories: 设计模式
tags:
- 设计模式
---
责任链模式是一种对象的行为模式。
## 责任链模式定义
责任链模式：避免请求发送者与接收者耦合在一起，让多个对象都有可能接收请求，将这些对象连接成一条链，并且沿着这条链传递请求，直到有对象处理它为止。

在责任链模式里，很多的对象由每一个对象对其下家的引用而联接起来形成一条链。请求在这个链上传递，直到链上的某一个对象决定处理此请求。发出这个请求的客户端并不知道链上的哪一个对象最终处理这个请求，这使得系统可以在不影响客户端的情况下动态地重新组织链和分配责任。比如servlet中的filter，有多层，当然，这个在Spring Security使用的更加淋漓尽致。

在责任链模式中发出请求的客户端并不知道这当中的哪个对象最终处理这个请求，这样系统的更改可以在不影响客户端的情况下动态的重新组织和分配责任。

## 责任链模式的结构

![handlerchain](http://ovcjgn2x0.bkt.clouddn.com/handlerchain.png "责任链模式")

责任链模式中，主要的角色有Handler和ConcreteHandler：

- `Handler`定义一个处理请求的接口；
- `ConcreteHandler`具体处理类，处理它所负责的请求，可访问它的后继者。如果可以处理该请求，就处理，否则就将该请求转发给它的后继者。

## 责任链模式的实现
抽象类Handler，具体的实现都要继承该抽象类。定义了继任者，以及处理请求的方法。

```java
@Data
public abstract class Handler {

    protected Handler successor;//继任者
    //处理请求
    public void handleRequest(int num) {}
}
```
第一个继任者，num小于10则处理，否则继续传递。

```java
public class ConcreteHandler1 extends Handler {

    public void handleRequest(int num) {
        if (num >= 0 && num < 10) {
            System.out.println(this.getClass() + "  处理请求  " + num);
        } else if (successor != null) {
            successor.handleRequest(num);
        }
    }
}
```
第二个继任者，处理10到20的num。

```java
public class ConcreteHandler2 extends Handler {

    public void handleRequest(int num) {
        if (num >= 10 && num < 20) {
            System.out.println(this.getClass() + "  处理请求  " + num);
        } else if (successor != null) {
            successor.handleRequest(num);
        }
    }
}
```
第三个继任者，处理20到30的num。

```java
public class ConcreteHandler3 extends Handler {

    public void handleRequest(int num) {
        if (num >= 20 && num < 30) {
            System.out.println(this.getClass() + "  处理请求  " + num);
        } else if (successor != null) {
            successor.handleRequest(num);
        }
    }
}
```
客户端示例代码。调用链为：1->2->3。

```java
public class TestDemo {

    public static void main(String[] args) {
        Handler handler1 = new ConcreteHandler1();
        Handler handler2 = new ConcreteHandler2();
        Handler handler3 = new ConcreteHandler3();

        //设置责任链的前驱和后继
        handler1.setSuccessor(handler2);
        handler2.setSuccessor(handler3);

        handler1.handleRequest(15);
    }
}
```

## 总结
本文主要介绍了行为模式的责任链模式。在看到一些文章处理很多`if...else...`语句时，也会用责任链模式处理，将条件判断分散到各个实现类中，并且这些处理类的优先处理顺序可以随意的设定，并且如果想要添加新的 handler 类也是十分简单的，这符合开放闭合原则。不过需要注意避免责任链中出现循环引用。优缺点总结如下：

优点：

- 降低耦合度。它将请求的发送者和接收者解耦。
- 简化了对象。使得对象不需要知道链的结构。增加新的请求处理类很方便。
- 增强给对象指派职责的灵活性。通过改变链内的成员或者调动它们的次序，允许动态地新增或者删除责任。

不足之处：

- 不能保证请求一定被接收。
- 系统性能将受到一定影响，可能会造成循环调用。可能不容易观察运行时的特征，有碍于除错。

### 参考
1. [话设计模式—责任链模式](http://blog.csdn.net/lmb55/article/details/51052866)
2. [设计模式之责任链模式](https://www.jianshu.com/p/d22f4e5408f5)
