---
title: 比较Spring AOP与AspectJ
date: 2018-1-24 
categories: java
tags:
- Spring
- Java
---
> 本文翻译自博客[Comparing Spring AOP and AspectJ](http://www.baeldung.com/spring-aop-vs-aspectj)

## 介绍
如今有多个可用的AOP库，这些组件需要回答一系列的问题：

- 是否与我现有的应用兼容？
- 我在哪实现AOP？
- 集成到我的应用是否很快？
- 性能开销是多少？

本文中，我们将会着重回答这些问题，并介绍两款Java最流行的AOP框架：Spring AOP 和 AspectJ。

## AOP概念
在我们开始之前，让我们对术语和核心概念做一个快速，高水平的回顾：

- Aspect切面：一个分布在应用程序中多个位置的标准代码/功能，通常与实际的业务逻辑（例如事务管理）不同。 每个切面都侧重于一个特定的横切功能。
- Joinpoint连接点：这是程序执行中的特定点，如方法执行，构调用造函数或字段赋值等。
- Advice通知：在一个连接点中，切面采取的行动
- Pointcut切点：一个匹配连接点的正则表达式。 每当任何连接点匹配一个切入点时，就执行与该切入点相关联的指定通知。
- Weaving织入：链接切面和目标对象来创建一个通知对象的过程。

## Spring AOP and AspectJ
现在，一起来讨论Spring AOP and AspectJ，跨越多指标，如能力和目标、织入方式、内部结构、连接点和简单性。

### Capabilities and Goals
简而言之，Spring AOP和AspectJ有不同的目标。   
Spring AOP旨在通过Spring IoC提供一个简单的AOP实现，以解决编码人员面临的最常出现的问题。这并不是完整的AOP解决方案，它只能用于Spring容器管理的beans。

另一方面，AspectJ是最原始的AOP实现技术，提供了玩这个的AOP解决方案。AspectJ更为健壮，相对于Spring AOP也显得更为复杂。值得注意的是，AspectJ能够被应用于所有的领域对象。

### Weaving
 AspectJ and Spring AOP使用了不同的织入方式，这影响了他们在性能和易用性方面的行为。   
 AspectJ使用了三种不同类型的织入：
 
 1. 编译时织入：AspectJ编译器同时加载我们切面的源代码和我们的应用程序，并生成一个织入后的类文件作为输出。
 2. 编译后织入：这就是所熟悉的二进制织入。它被用来编织现有的类文件和JAR文件与我们的切面。
 3. 加载时织入：这和之前的二进制编织完全一样，所不同的是织入会被延后，直到类加载器将类加载到JVM。

更多关于AspectJ的信息，请见[head on over to this article](http://www.baeldung.com/aspectj)。

AspectJ使用的是编译期和类加载时进行织入，Spring AOP利用的是运行时织入。

运行时织入，在使用目标对象的代理执行应用程序时，编译这些切面（使用JDK动态代理或者CGLIB代理）。

![springaop-process](http://ovcjgn2x0.bkt.clouddn.com/springaop-process.png "springaop-process")

### Internal Structure and Application
Spring AOP 是一个基于代理的AOP框架。这意味着，要实现目标对象的切面，将会创建目标对象的代理类。这可以通过下面两种方式实现：

- JDK动态代理：Spring AOP的首选方法。 每当目标对象实现一个接口时，就会使用JDK动态代理。
- CGLIB代理：如果目标对象没有实现接口，则可以使用CGLIB代理。

关于Spring AOP可以通过官网了解更多。   
另一方面，AspectJ在运行时不做任何事情，类和切面是直接编译的。因此，不同于Spring AOP，他不需要任何设计模式。织入切面到代码中，它引入了自己的编译期，称为AspectJ compiler (ajc)。通过它，我们编译应用程序，然后通过提供一个小的（<100K）运行时库运行它。

### Joinpoints
在上一小节，我们介绍了Spring AOP基于代理模式。因此，它需要目标类的子类，并相应的应用横切关注点。但是也伴随着局限性，我们不能跨越“final”的类来应用横切关注点（或切面），因为它们不能被覆盖，从而导致运行时异常。

同样地，也不能应用于静态和final的方法。由于不能覆写，Spring的切面不能应用于他们。因此，Spring AOP由于这些限制，只支持执行方法的连接点。   
然而，AspectJ在运行前将横切关注点直接织入实际的代码中。 与Spring AOP不同，它不需要继承目标对象，因此也支持其他许多连接点。AspectJ支持如下的连接点：
![joinpoint](http://ovcjgn2x0.bkt.clouddn.com/joinpoint.jpg "连接点支持")

同样值得注意的是，在Spring AOP中，切面不适用于同一个类中调用的方法。这很显然，当我们在同一个类中调用一个方法时，我们并没有调用Spring AOP提供的代理的方法。如果我们需要这个功能，可以在不同的beans中定义一个独立的方法，或者使用AspectJ。

### Simplicity
Spring AOP显然更加简单，因为它没有引入任何额外的编译期或在编译期织入。它使用了运行期织入的方式，因此是无缝集成我们通常的构建过程。尽管看起来很简单，Spring AOP只作用于Spring管理的beans
。

然而，使用AspectJ，我们需要引入AJC编译器，重新打包所有库（除非我们切换到编译后或加载时织入）。这种方式相对于前一种，更加复杂，因为它引入了我们需要与IDE或构建工具集成的AspectJ Java工具（包括编译器（ajc），调试器（ajdb），文档生成器（ajdoc），程序结构浏览器（ajbrowser））。

### Performance
考虑到性能问题，编译时织入比运行时织入快很多。Spring AOP是基于代理的框架，因此应用运行时会有目标类的代理对象生成。另外，每个切面还有一些方法调用，这会对性能造成影响。

AspectJ不同于Spring AOP，是在应用执行前织入切面到代码中，没有额外的运行时开销。

由于以上原因，AspectJ经过测试大概8到35倍快于Spring AOP。[benchmarks](https://web.archive.org/web/20150520175004/https://docs.codehaus.org/display/AW/AOP+Benchmark)
## 对比
这个快速表总结了Spring AOP和AspectJ之间的主要区别：
![com](http://ovcjgn2x0.bkt.clouddn.com/aj-vs-aop.jpg "对比")

## 选择合适的框架

如果我们分析本节所有的论点，我们就会开始明白，没有绝对的一个框架比另一个框架更好。   
简而言之，选择很大程度上取决我们的需求：

- 框架：如果应用程序不使用Spring框架，那么我们别无选择，只能放弃使用Spring AOP的想法，因为它无法管理任何超出spring容器范围的东西。 但是，如果我们的应用程序完全是使用Spring框架创建的，那么我们可以使用Spring AOP，因为它很直接便于学习和应用。
- 灵活性：鉴于有限的连接点支持，Spring AOP并不是一个完整的AOP解决方案，但它解决了程序员面临的最常见的问题。 如果我们想要深入挖掘并利用AOP达到其最大能力，并希望获得来自各种可用连接点的支持，那么AspectJ是最佳选择。
- 性能：如果我们使用有限的切面，那么性能差异很小。 但是，有时候应用程序有数万个切面的情况。 在这种情况下，我们不希望使用运行时织入，所以最好选择AspectJ。 已知AspectJ比Spring AOP快8到35倍。
- 共同优点：这两个框架是完全兼容的。 我们可以随时利用Spring AOP，并且仍然使用AspectJ来获得前者不支持的连接点。

## 总结
在这篇文章中，我们分析了Spring AOP和AspectJ比较关键的几个方面。我们比较了AOP和AOP两种方法的灵活性，以及它们与我们的应用程序的匹配程度。


