---
title: 设计模式之观察者模式
date: 2017-03-15
categories: 设计模式
tags:
- 设计模式
---
观察者模式属于行为模式。
## 观察者模式的定义
观察者模式又称为发布/订阅模式，是一种对象的行为型模式。它定义了对象之间的一对多的依赖关系，使得每当一个对象状态发生改变时，其相关依赖对象都得到通知并被自动更新。观察者模式的优点在于实现了表示层和数据层的分离，并定义了稳定的更新消息传递机制，类别清晰，抽象了更新接口，使得相同的数据层可以有各种不同的表示层。
![obeserve.png](http://ovcjgn2x0.bkt.clouddn.com/obeserve.png "观察者模式")
使用场景：

- 对一个对象的修改涉及对其它对象的修改，而且不知道有多少对象需要进行相应修改。
- 对象应该能够在不用假设对象标识的前提下通知其它对象。
## 观察者模式的结构
在观察者模式中，包括以下四个角色：

- 主题（Subject）：主题是一个接口，该接口规定了具体主题需要实现的方法，比如，添加、删除观察者以及通知观察者更新数据的方法。
- 观察者（Observer）：观察者是一个接口，该接口规定了具体观察者用来更新数据的方法。
- 具体主题（ConcreteSubject）：具体主题是实现主题接口类的一个实例，该实例包含有可以经常发生变化的数据。具体主题需使用一个集合，比如ArrayList，存放观察者的引用，以便数据变化时通知具体观察者。
- 具体观察者（ConcreteObserver）：具体观察者是实现观察者接口类的一个实例。具体观察者包含有可以存放具体主题引用的主题接口变量，以便具体观察者让具体主题将自己的引用添加到具体主题的集合中，使自己成为它的观察者，或让这个具体主题将自己从具体主题的集合中删除，使自己不再是它的观察者。

## 观察者模式的实现
定义一个抽象被观察者接口，抽象主题，可以增加和删除观察者角色。

```java
public interface Observerable {
    public void registerObserver(Observer o);
    public void removeObserver(Observer o);
    public void notifyObserver();   
}
```
定义一个抽象观察者接口，订阅消息的更新：

```java
public interface Observer {
    public void update(String message);
}
```
具体的主题，实现了Observerable接口，对Observerable接口的三个方法进行了具体实现，同时有一个List集合，用以保存注册的观察者，等需要通知观察者时，遍历该集合即可。

```java
public class WechatServer implements Observerable {
    
    //注意到这个List集合的泛型参数为Observer接口，设计原则：面向接口编程而不是面向实现编程
    private List<Observer> list;
    private String message;
    
    public WechatServer() {
        list = new ArrayList<Observer>();
    }
    
    @Override
    public void registerObserver(Observer o) {
        list.add(o);
    }
    
    @Override
    public void removeObserver(Observer o) {
        if(!list.isEmpty())
            list.remove(o);
    }

    //遍历
    @Override
    public void notifyObserver() {
        for(int i = 0; i < list.size(); i++) {
            Observer oserver = list.get(i);
            oserver.update(message);
        }
    }
    
    public void setInfomation(String s) {
        this.message = s;
        System.out.println("更新消息： " + s);
        //消息更新，通知所有观察者
        notifyObserver();
    }

}
```
定义具体观察者，微信公众号的订阅用户User：

```java
public class User implements Observer {

    private String name;
    private String message;
    
    public User(String name) {
        this.name = name;
    }
    
    @Override
    public void update(String message) {
        this.message = message;
        read();
    }
    
    public void read() {
        System.out.println(name + " 收到推送消息： " + message);
    }   
}
```
测试类：

```java
public class Test {
    
    public static void main(String[] args) {
        WechatServer server = new WechatServer();
        
        Observer user1 = new User("user1");
      
        server.registerObserver(user1);
        server.setInfomation("test");
    }
}
```
我们注册了user1，并发布了一条消息`test`，user1能够正常收到该消息通知。
## 总结
观察者模式将观察者和主题（被观察者）彻底解耦，主题只知道观察者实现了某一接口，即Observer接口。并不需要观察者的具体类是谁、做了些什么或者其他任何细节。任何时候我们都可以增加新的观察者。因为主题唯一依赖的东西是一个实现了Observer接口的对象列表。

具体主题和具体观察者是松耦合关系。由于主题接口仅仅依赖于观察者接口，因此具体主题只是知道它的观察者是实现观察者接口的某个类的实例，但不需要知道具体是哪个类。同样，由于观察者仅仅依赖于主题接口，因此具体观察者只是知道它依赖的主题是实现主题接口的某个类的实例，但不需要知道具体是哪个类。

观察者模式满足“开-闭原则”。主题接口仅仅依赖于观察者接口。这样，就可以让创建具体主题的类也仅仅是依赖于观察者接口，因此，如果增加新的实现观察者接口的类，不必修改创建具体主题的类的代码。同样，创建具体观察者的类仅仅依赖于主题接口，如果增加新的实现主题接口的类，也不必修改创建具体观察者类的代码。

### vs 发布/订阅模式
在翻阅资料的时候，有人把观察者（Observer）模式等同于发布（Publish）/订阅（Subscribe）模式，也有人认为这两种模式还是存在差异。发布/订阅模式中，订阅者把自己想订阅的事件注册到调度中心，当该事件触发时候，发布者发布该事件到调度中心（顺带上下文），由调度中心统一调度订阅者注册到调度中心的处理代码。

虽然两种模式都存在订阅者和发布者（具体观察者可认为是订阅者、具体目标可认为是发布者），但是观察者模式是由具体目标调度的，而发布/订阅模式是统一由调度中心调的，所以观察者模式的订阅者与发布者之间是存在依赖的，而发布/订阅模式则不会。

### 参考
1. [JAVA设计模式之观察者模式](https://www.cnblogs.com/luohanguo/p/7825656.html)
2. [设计模式（三）：观察者模式与发布/订阅模式区别](https://www.cnblogs.com/lovesong/p/5272752.html)
