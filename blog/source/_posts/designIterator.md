---
title: 设计模式之迭代子模式
date: 2017-03-28
categories: 设计模式
tags:
- 设计模式
---
迭代子模式又叫游标(Cursor)模式，是对象的行为模式。
## 迭代子模式的定义
迭代子模式可以顺序地访问一个聚集中的元素而不必暴露聚集的内部表象。我们常见的集合有很多种类，其顶层数据存储和组织方式的不同导致了我们在对数据进行遍历的时候存在一些差异，迭代器模式就是通过实现某种统一的方式来实现对不同的集合的遍历，同时又不暴露出其底层的数据存储和组织方式。

例如，如果没有使用Iterator，遍历一个数组的方法是使用索引： 

```java
for(int i=0; i<array.size(); i++) { 
	get(i)
} 
```
而访问一个链表（LinkedList）又必须使用while循环： 

```java
while((e=e.next())!=null) { 
	 e.data()
} 
```
以上两种方法客户端都必须事先知道集合的内部结构，访问代码和集合本身是紧耦合，无法将访问逻辑从集合类和客户端代码中分离出来，每一种集合对应一种遍历方法，客户端代码无法复用。 

更恐怖的是，如果以后需要把ArrayList更换为LinkedList，则原来的客户端代码必须全部重写。 


为解决以上问题，Iterator模式总是用同一种逻辑来遍历集合： 

```java
for(Iterator it = yourCollection.iterater(); it.hasNext(); ) 
{ ... } 
```

客户端自身不维护遍历集合的"指针"，所有的内部状态（如当前元素位置，是否有下一个元素）都由`Iterator`来维护，而这个`Iterator`由集合类通过工厂方法生成，因此，它知道如何遍历整个集合。客户端从不直接和集合类打交道，它总是控制Iterator，向它发送"向前"，"向后"，"取当前元素"的命令，就可以间接遍历整个集合。

## 迭代子模式的结构
迭代模式中有如下的角色：
![iteratorconstruct](http://ovcjgn2x0.bkt.clouddn.com/iteratorconstruct.jpg "迭代子模式的结构")

- 迭代器角色（Iterator）: 负责定义访问和遍历元素的接口。
- 具体迭代器角色（Concrete Iterator）：实现迭代器接口，并要记录遍历中的当前位置。
- 容器角色(Container):  负责提供创建具体迭代器角色的接口。
- 具体容器角色（Concrete Container）：实现创建具体迭代器角色的接口， 这个具体迭代器角色与该容器的结构相关。

## 迭代子模式的实现
迭代器接口实现，定义了获取第一个节点的方法，前一个节点和后一个节点，以及判断是否有下一个节点。

```java
public interface Iterator {
    public Object first();
    
    public Object previous();
    
    public Object next();

    public boolean hasNext();
}
```
具体实现迭代器，实现上述接口定义的方法。

```java
public class MyIterator implements Iterator{
    private List<Object> list;
    private int index = 0;

    public MyIterator(List<Object> list) {
        this.list = list;
    }
    @Override
    public Object previous() {
        if((this.index - 1) < 0){
            return null;
        }else{
            return this.list.get(--index);
        }
        
    }

    @Override
    public Object next() {
        if((this.index + 1) >= this.list.size()){
            return null;
        }else{
            return this.list.get(++index);
        }
    }
    @Override
    public boolean hasNext() {
        if(this.index < (this.list.size() - 1)){
            return true;
        }
        return false;
    }

    @Override
    public Object first() {
        if(this.list.size() <= 0){
            return null;
        }else{
            return this.list.get(0);
        }
    }
}
```
容器定义，定义了两个抽象方法，用来设置具体的迭代器实现以及注入容器中的元素。

```java
public abstract class Container {

    public abstract Iterator iterator();
    
    public abstract void put(Object obj);
}
```
具体的容器类基于List，实现抽象方法。

```java
public class MyContainer extends Container{
    private List<Object> list;
    
    public MyContainer() {
        this.list = new ArrayList<Object>();
    }
    @Override
    public void put(Object obj){
        this.list.add(obj);
    }
    @Override
    public Iterator iterator() {
        return new MyIterator(list);
    }
}
```
客户端测试类。设置元素，并使用迭代器进行遍历。

```java
public class ClientTest {

	public static void main(String[] args) {

		//创建一个自定义容器，直接使用ArrayList的实现
		Container strContainer = new MyContainer();
		strContainer.put("001");
		strContainer.put("002");

		Iterator myIterator = strContainer.iterator();

		//使用迭代器遍历
		System.out.println(myIterator.first());
		while (myIterator.hasNext()) {
			System.out.println(myIterator.next());
		}
	}
}
```

![iterator](http://ovcjgn2x0.bkt.clouddn.com/iterator.jpg "类图")

## 总结
Iterator模式是用于遍历集合类的标准访问方法。它可以把访问逻辑从不同类型的集合类中抽象出来，从而避免向客户端暴露集合的内部结构。 

适用场景：

- 访问一个聚合对象的内容而无须暴露它的内部表示。
- 需要为聚合对象提供多种遍历方式。
- 为遍历不同的聚合结构提供一个统一的接口。

优点：

- 它支持以不同的方式遍历一个聚合对象。
- 迭代器简化了聚合类。在同一个聚合上可以有多个遍历。
- 在迭代器模式中，增加新的聚合类和迭代器类都很方便，无须修改原有代码。
- 系统需要访问一个聚合对象的内容而无需暴露它的内部表示。

缺点：

- 由于迭代器模式将存储数据和遍历数据的职责分离，增加新的聚合类需要对应增加新的迭代器类，类的个数成对增加，这在一定程度上增加了系统的复杂性。
- 迭代器模式在遍历的同时更改迭代器所在的集合结构会导致出现异常。所以使用foreach语句只能在对集合进行遍历，不能在遍历的同时更改集合中的元素。

### 参考
1. [java设计模式----迭代子模式](https://www.cnblogs.com/JAYIT/p/5081695.html)
2. [Head First设计模式之迭代器模式](https://www.cnblogs.com/xcsn/p/7499521.html)
