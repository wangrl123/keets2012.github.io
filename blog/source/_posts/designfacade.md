---
title: 设计模式之外观模式
date: 2017-03-3
categories: 设计模式
tags:
- 设计模式
---
## 1. 概述
外观模式通过外观的包装，使复杂的系统对外只能看到外观对象，而不会看到具体的细节对象，为子系统中的一组接口提供了一个统一的访问接口，这个接口使得子系统更容易被访问或者使用。 这样无疑会降低应用程序的复杂度，并且提高了程序的可维护性。

## 2. 组成
为子系统中的一组接口提供一个一致的界面， Facade模式定义了一个高层接口，这个接口使得这一子系统更加容易使用。引入外观角色之后，用户只需要直接与外观角色交互，用户与子系统之间的复杂关系由外观角色来实现，从而降低了系统的耦合度。
![facade](http://ovcjgn2x0.bkt.clouddn.com/facade.png "facade模式")

其组成如下：

- 外观角色（Facade）：是核心，他被客户client角色调用，知道各个子系统的功能。同时根据客户角色已有的需求预订了几种功能组合。
- 子系统角色（Subsystem classes）：实现子系统的功能，并处理由Facade对象指派的任务。对子系统而言，facade和client角色是未知的，没有Facade的任何相关信息；即没有指向Facade的实例。
- 客户角色（client）：调用facade角色获得完成相应的功能。
## 3. 示例

### 3.1 子系统

```java
	//子系统A
    public class SubSystemOne
    {
        public void MethodeOne()
        {
            System.out.println("Sub System A method.");
        }
    }

    /// 子系统B
    public class SubSystemTwo
    {
        public void MethodTwo()
        {
            System.out.println("Sub System B method.");
        }
    }

    // 子系统C
    public class SubSystemThree
    {
        public void MethodThree()
        {
            System.out.println("Sub System C method.");
        }
    }
```

一共有三个子系统，每个子系统都有自己的方法。

### 3.2 外观类
```java
    // 外观类
    public class Facade
    {
        private SubSystemOne one;
        private SubSystemTwo two;
        private SubSystemThree three;

        public Facade()
        {
            one = new SubSystemOne();
            two = new SubSystemTwo();
            three = new SubSystemThree();
        }

        public void MethodA()
        {
            System.out.println("\nMethod group A----");
            one.MethodeOne();
            two.MethodTwo();
        }

        public void MethodB()
        {
            System.out.println("\nMethod group B----");
            two.MethodTwo();
            three.MethodThree();
        }
    }
```

外观类是提供给客户端调用直接进行调用，外观类对子系统提供的方法进行组合，形成两个组，分别表现为不同的行为（调用不同的方法）。
### 3.3 客户端

```java
    public class FacadeTest
    {
        public static void main(string[] args)
        {
            // 调用外观类
            Facade facade = new Facade();
            facade.MethodA();
            facade.MethodB();
        }
    }
```

由于Facade的作用，客户端可以根本不知道子系统类的存在。

## 4. 场景和优缺点
### 4.1 使用场景

- 当你要为一个复杂子系统提供一个简单接口时。子系统往往因为不断演化而变得越来越复杂。大多数模式使用时都会产生更多更小的类。
    这使得子系统更具可重用性，也更容易对子系统进行定制，但这也给那些不需要定制子系统的用户带来一些使用上的困难。facade可以提供一个简单的缺省视图，
    这一视图对大多数用户来说已经足够，而那些需要更多的可定制性的用户可以越过facade层。
- 客户程序与抽象类的实现部分之间存在着很大的依赖性。引入 facade将这个子系统与客户以及其他的子系统分离，可以提高子系统的独立性 和可移植性。
- 当你需要构建一个层次结构的子系统时，使用 facade模式定义子系统中每层的入口点。如果子系统之间是相互依赖的，你可以让它们仅通过facade进行通讯，从而简化了它们之间的依赖关系。

### 4.2 优点

- Facade模式降低了客户端对子系统使用的复杂性。
- 外观模式松散了客户端与子系统的耦合关系，让子系统内部的模块能更容易扩展和维护。
- 通过合理使用Facade，可以帮助我们更好的划分访问的层次。
### 4.3 缺点

- 不能很好地限制客户使用子系统类，如果对客户访问子系统类做太多的限制则减少了可变性和灵活性。
- 在不引入抽象外观类的情况下，增加新的子系统可能需要修改外观类或客户端的源代码，违背了“开闭原则”。

## 5. 总结
本文主要讲了设计模式的外观模式，从概述到组成，以一个实例进行讲解各个组成角色，最后概括了使用场景和外观模式的优缺点。外观模式的思路，有点让人联想其适配器模式，适配器模式是将一个接口通过适配来间接转换为另一个接口。而外观模式，其主要是提供一个整洁的一致的接口给客户端。

外观模式要求一个子系统的外部与其内部的通信通过一个统一的外观对象进行，外观类将客户端与子系统的内部复杂性分隔开，使得客户端只需要与外观对象打交道，而不需要与子系统内部的很多对象打交道。从很大程度上提高了客户端使用的便捷性，使得客户端无须关心子系统的工作细节，通过外观角色即可调用相关功能。

另外，不要试图通过外观类为子系统增加新行为，不要通过继承一个外观类在子系统中加入新的行为，这种做法是错误的。外观模式的用意是为子系统提供一个集中化和简化的沟通渠道，而不是向子系统加入新的行为，新的行为的增加应该通过修改原有子系统类或增加新的子系统类来实现，不能通过外观类来实现。


### 参考
[设计模式--外观模式Facade](http://blog.csdn.net/hguisu/article/details/7533759)

