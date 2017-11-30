---
title: 设计模式之建造者模式
date: 2017-02-26
categories: 设计模式
tags:
- 设计模式
---
## 1. 名词解释
> 将一个复杂对象的构建与它的表示分离，使得同样的构建过程可以创建不同的表示。

比如一台电脑包括主机、显示器、键盘等外设，这些部件组成了完整的一台电脑。如何将这些部件组装成一台完整的电脑并返回给用户，这是建造者模式需要解决的问题。建造者模式（builder）又称为生成器模式，从名词就可以看出，它是一种较为复杂、使用频率也相对较低的创建型模式。建造者模式为客户端返回的不是一个简单的产品，而是一个由多个部件组成的复杂产品。 
 

## 2. 建造者模式UML图

![builder](http://ovcjgn2x0.bkt.clouddn.com/builder2.jpg "builder模式")

上图中包含了建造者模式的四个主要角色：

- Builder（抽象建造者）：它为创建一个产品Product对象的各个部件指定抽象方法，在该接口中一般声明两类方法，一类方法是buildPartX()，它们用于创建复杂对象的各个部件；另一类方法是getResult()，它们用于返回复杂对象。Builder既可以是抽象类，也可以是接口。
- ConcreteBuilder（具体建造者）：它实现了Builder抽象方法，实现各个部件的具体构造和装配方法，定义并明确它所创建的复杂对象，也可以提供一个方法返回创建好的复杂产品对象。依赖于Product。
- Product（产品角色）：它是被构建的复杂对象，包含多个组成部件，具体建造者创建该产品的内部表示并定义它的装配过程。
- Director（指挥者）：指挥者又称为导演类，它负责安排复杂对象的建造次序，指挥者与抽象建造者之间存在关联关系，可以在其construct()建造方法中调用建造者对象的部件构造与装配方法，完成复杂对象的建造。客户端一般只需要与指挥者进行交互，在客户端确定具体建造者的类型，并实例化具体建造者对象（也可以通过配置文件和反射机制），然后通过指挥者类的构造函数或者Setter方法将该对象传入指挥者类中。

## 3. 建造者模式实现

下面代码基于Java实现的一个建造者模式。

```java
@data
class Product {  
    private String name;  
    private String type;  
    public void showProduct(){  
        System.out.println("名称："+name);  
        System.out.println("型号："+type);  
    }  
}  

//抽象类
abstract class Builder {  
    public abstract void buildPart(String arg1, String arg2);  
    public abstract Product getProduct();  
}  

//具体建造者
class ConcreteBuilder extends Builder {  
    private Product product = new Product();  
      
    public Product getResult() {  
        return product;  
    }  
  
    public void buildPart(String arg1, String arg2) {  
        product.setName(arg1);  
        product.setType(arg2);  
    }  
}  

// 指挥者 
public class Director {  
    private Builder builder;
  
    public Director(Builder builder) {
       this.builder=builder;
    }
      
    public void setBuilder(Builder builder) {
       this.builder=builer;
    } 
    public Product construct(){  
        builder.buildPart("lenvono","Y470");  
        return builder.getResult();  
    }   
}  
//客户端调用
public class Client {  
    public static void main(String[] args){ 
        Builder builder = new ConcreteBuilder(); 
        Director director = new Director(builder);  
        Product product = director.construct();  
        product1.showProduct();  
    }  
}

```
四个角色都包含在上面的实现中，客户端进行调用，对于客户端而言，只需关心具体的建造者即可。在指挥者类中可以注入一个抽象建造者类型的对象，其核心在于提供了一个建造方法construct()，在该方法中调用了builder对象的构造部件的方法，最后返回一个产品对象。
      
## 4. 与工厂模式区别
建造者模式的优点是封装性好，且易于扩展。上面提到，对于客户端而言，只需关心具体的建造者即可。使用建造者模式可以优先的封装变化，product和builder比较稳定，主要的业务逻辑封装在控制类中对整体可取得比较好的稳定性。如需扩展，只需要加一个新的建造者，对之前代码没有影响。

与工厂模式相比，建造者模式一般用来创建更为复杂的对象，因为对象的创建过程更为复杂，因此将对象的创建过程独立出来组成一个新的类——指挥者类。也就是说，工厂模式是将对象的全部创建过程封装在工厂类中，由工厂类向客户端提供最终的产品；而建造者模式中，建造者类一般只提供产品类中各个组件的建造，而将具体建造过程交付给指挥者类。由指挥者类负责将各个组件按照特定的规则组建为产品，然后将组建好的产品交付给客户端。

## 5. 总结
本文讲解了设计模式中的建造者模式，大的分类属于创建型模式。首先介绍了建造者模式的概念；然后给出了建造者模式的类图，并对其中涉及到的四个角色进行了解释；又给出了基于Java实现的代码；最后简单说了下其优点，与工厂模式的区别。建造者模式主要适用于创建一些复杂的对象，这些对象的内部组成构件间的建造顺序是稳定的，但是对象的内部组成构件面临着复杂的变化。


---
### 参考
1. 《java与模式》阎宏
2.  [23种设计模式（4）：建造者模式](http://blog.csdn.net/zhengzhb/article/details/7375966)

