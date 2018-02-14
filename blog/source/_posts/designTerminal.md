---
title: 设计模式之命令模式
date: 2017-03-31
categories: 设计模式
tags:
- 设计模式
---
命令模式是一种行为模式。
## 命令模式的定义
命令模式是一个高内聚的模式，将一个请求封装成一个对象，从而让你使用不同的请求把客户端参数化，对请求排队或者记录日志，可以提供命令的撤销和恢复功能。命令模式的核心在于引入了命令类，通过命令类来降低发送者和接收者的耦合度，请求发送者只需指定一个命令对象，再通过命令对象来调用请求接收者的处理方法。

![Command](http://img.my.csdn.net/uploads/201304/15/1366033467_9048.jpg "命令模式")

命令模式可以将请求发送者和接收者完全解耦，发送者与接收者之间没有直接引用关系，发送请求的对象只需要知道如何发送请求，而不必知道如何完成请求。


## 命令模式的结构
命令模式包含如下角色：

- Receiver接收者角色   
命令接收者模式，命令传递到这里执行对应的操作。

- Command命令角色    
需要执行的命令都在这里声明

- Invoker调用者角色    
接收到命令，并执行命令，也就是命令的发动者和调用者
## 命令模式的实现
抽象Command类。

```java
public abstract class Command {  
    public abstract void execute();  
}
```
调用者Invoker类。

```java
public class Invoker {  
    private Command command;  
      
    //构造注入  
    public Invoker(Command command) {  
        this.command = command;  
    }  
      
    //设值注入  
    public void setCommand(Command command) {  
        this.command = command;  
    }  
      
    //业务方法，用于调用命令类的execute()方法  
    public void call() {  
        command.execute();  
    }  
}  
```
具体Command类，设置对应的接收者。

```java
public  class ConcreteCommand extends Command {  
    private Receiver receiver; //维持一个对请求接收者对象的引用  
  
    public ConcreteCommand(Receiver receiver) {
        super();
        this.receiver = receiver;
    }

    public void execute() {  
        receiver.action(); //调用请求接收者的业务处理方法action()  
    }  
}  
```
抽象接收者。

```java
public  abstract class Receiver {  
    public abstract void action();
}  
```
第一个接收者。

```java
public class ConcreteReceiver extends Receiver{

    @Override
    public void action() {
        System.out.println("ConcreteReceiver receives the command!");
    }

}
```
第二个接收者。

```java
public class ConcreteReceiver2 extends Receiver{

    @Override
    public void action() {
        System.out.println("ConcreteReceiver2 receives the command!");
    }

}
```
测试类，创建了连个接收者，分别发送指令。

```java
public class Client {

    public static void main(String[] args) {
        Receiver receiver1 = new ConcreteReceiver();
        Command command1 = new ConcreteCommand(receiver1);

        Invoker invoker = new Invoker();
        invoker.setCommand(command1);
        invoker.call();

        Receiver receiver2 = new ConcreteReceiver2();
        Command command2 = new ConcreteCommand(receiver2);

        invoker.setCommand(command2);
        invoker.call();
    }
}
```
## 总结

命令模式的本质是对请求进行封装，一个请求对应于一个命令，将发出命令的责任和执行命令的责任分割开。每一个命令都是一个操作：请求的一方发出请求要求执行一个操作；接收的一方收到请求，并执行相应的操作。命令模式允许请求的一方和接收的一方独立开来，使得请求的一方不必知道接收请求的一方的接口，更不必知道请求如何被接收、操作是否被执行、何时被执行，以及是怎么被执行的。命令模式的关键在于引入了抽象命令类，请求发送者针对抽象命令类编程，只有实现了抽象命令类的具体命令才与请求接收者相关联。

命令模式的优点是：

- 类间解藕   
调用者角色与接受者角色之间没有任何依赖关系，调用者实现功能时只需要调用Command抽象类的execute方法就可以，不需要知道到底是哪个接收者执行。

- 可扩展性   
Command子类可以非常容易的扩展，而调用者Invoker和高层次的模块Client不产生严重的代码藕合

其缺点是造成了类膨胀，如果有多个子命令对应多个Command子类，比较繁琐。

### 参考
1. [java设计模式之命令模式](https://www.cnblogs.com/lfxiao/p/6825644.html)
2. [设计模式之命令模式---Command Pattern](http://blog.csdn.net/hfreeman2008/article/details/52134841)

