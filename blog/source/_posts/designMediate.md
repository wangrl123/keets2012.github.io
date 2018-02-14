---
title: 设计模式之中介者模式
date: 2017-04-3
categories: 设计模式
tags:
- 设计模式
---
中介者模式属于行为型模式。
## 中介者模式的定义
定义：用一个中介者对象来封装一系列的对象交互。中介者使得各对象不需要显式地相互引用，从而使其松散耦合，而且可以独立地改变它们之间的交互。

![mediator](http://ovcjgn2x0.bkt.clouddn.com/mediator.png "中介者模式")
在软件开发中，通过提供一个统一的接口让系统不同部分进行通信。一般，如果系统有很多子模块需要直接沟通，都要创建一个中央控制点让其各模块通过中央控制点进行交互。中介者模式可以让这些子模块不需要直接沟通，从而达到进行解耦的目的。在现实生活中，有很多中介者模式的身影，例如二手车平台、婚姻中介和房产中介，通过平台方介入进行协调供需之间的关系。

## 中介者模式的结构
中介者模式有如下角色：

- Mediator   
中介者定义一个接口用于与各同事（Colleague）对象通信。
- ConcreteMediator   
具体中介者通过协调各同事对象实现协作行为，了解并维护它的各个同事。
- Colleague   
抽象同事类。
- Colleagueclass   
具体同事类。每个具体同事类都只需要知道自己的行为即可，但是他们都需要认识中介者


## 中介者模式的实现
定义抽象Mediator：

```java
public abstract class Mediator {
	public abstract void notice(String content,Colleague coll);
}
```
定义抽象同事Colleague：

```java
public abstract class Colleague {
    protected String name;
    protected Mediator mediator;

    public Colleague(String name, Mediator mediator) {
        this.name = name;
        this.mediator = mediator;
    }
}
```
具体同事类继承自Colleague，此刻就可以与中介者mediator进行通信了。

```java
public class ColleagueA extends Colleague {
    public ColleagueA(String name, Mediator mediator) {
        super(name, mediator);
    }
    public void getNotice(String message){
        System.out.println("同事A"+name+"获得信息"+message);
    }
    //同事A与中介者通信
    public void contact(String message){
        mediator.notice(message, this);
    }
}
```

```java
public class ColleagueB extends Colleague {

    public ColleagueB(String name, Mediator mediator) {
        super(name, mediator);
    }
    public void getNotice(String message){
        System.out.println("同事B"+name+"获得信息"+message);
    }
    //同事B与中介者通信
    public void contact(String message){
        mediator.notice(message, this);
    }
}
```
定义具体中介者ConcreteMediator,具体中介者通过协调各同事对象实现协作行为，了解并维护它的各个同事。

```java
@Data
public class ConcreteMediator extends Mediator {
    ColleagueA collA;
    ColleagueB collB;

    @Override
    public void notice(String content, Colleague coll) {
        if (coll == collA) {
            collB.getNotice(content);
        } else {
            collA.getNotice(content);
        }
    }
}
```
定义中介者与具体同事类，中介者知晓每一个具体的Colleague类。

```java
public class Client {
    // 中介者，ColleagueA、ColleagueB
    public static void main(String[] args) {
        ConcreteMediator mediator = new ConcreteMediator();
        ColleagueA colleagueA = new ColleagueA("A", mediator);
        ColleagueB colleagueB = new ColleagueB("B", mediator);
        mediator.setCollA(colleagueA);
        mediator.setCollB(colleagueB);
        colleagueA.contact("我是A，我要联系B！");
        colleagueB.contact("我是B，收到A消息！");
    }
}
```
运行结果很简单，读者自己试一下吧。

## 总结
中介者模式简化了对象之间的关系，将系统的各个对象之间的相互关系进行封装，将各个同事类解耦，使得系统变为松耦合。并且提供系统的灵活性，使得各个同事对象独立而易于复用。

其缺点也很明显：   

- 中介者模式中，中介者角色承担了较多的责任，所以一旦这个中介者对象出现了问题，整个系统将会受到重大的影响。
- 新增加一个同事类时，不得不去修改抽象中介者类和具体中介者类，此时可以使用观察者模式和状态模式来解决这个问题。

中介者模式适用于当对象之间的交互变多时，为了防止一个类会涉及修改其他类的行为，可以使用中介者模式，将系统从网状结构变为以中介者为中心的星型结构。

### vs 外观模式   
外观模式主要是以封装和隔离为主要任务，中介者则是协调同事类之间的关系，因此，中介者具有部分业务的逻辑控制。他们之间的主要区别为： 

* 外观模式的子系统如果脱离外观模式还是可以运行的，而中介者模式增加了业务逻辑，同事类不能脱离中介者而独自存在。 
- 外观模式将子系统的逻辑隐藏，用户不知道子系统的存在，而中介者模式中，用户知道同事类的存在。

### 参考
1. [设计模式（十四）中介者模式](http://blog.csdn.net/itachi85/article/details/60466829)
2. [Java设计模式系列之中介者模式](https://www.cnblogs.com/ysw-go/p/5413958.html)
