---
title: 几种分布式调用链监控组件的实践与比较（一）实践
date: 2017-11-10
categories: Ops
tags:
- APM
- zipkin
- micro-service
---
引言：最近在调研与选型分布式调用链监控组件。选了主要的三种APM组件进行了实践与比较。本来打算一篇文章写完的，篇幅太长，打算分两篇。本文主要讲下链路traceing的基本概念和几种APM组件的实践，实践部分也没给出特别详细的步骤，因为本文重点不在具体的步骤。第二篇将会讲下几种APM选型的比较与性能测试。

## 1. 问题背景
微服务架构下，服务按照不同的维度进行拆分，一次请求请求往往需要涉及到多个服务。互联网应用构建在不同的软件模块集上，这些软件模块，有可能是由不同的团队开发、可能使用不同的编程语言来实现、有可能布在了几千台服务器，横跨多个不同的数据中心。因此，就需要一些可以帮助理解系统行为、用于分析性能问题的工具，以便发生故障的时候，能够快速定位和解决问题。

分布式调用链监控组件在这样的环境下产生了。最出名的是谷歌公开的论文提到的`Dapper`。开发Dapper是为了收集更多的复杂分布式系统的行为信息，然后呈现给Google的开发者们。这样的分布式系统有一个特殊的好处，因为那些大规模的低端服务器，作为互联网服务的载体，是一个特殊的经济划算的平台。想要在这个上下文中理解分布式系统的行为，就需要监控那些横跨了不同的应用、不同的服务器之间的关联动作。

市面上的APM（Application Performance Management）理论模型大多都是借鉴（borrow）`Google Dapper`论文，本文重点关注以下几种APM组件：

- Zipkin   
	由Twitter公司开源，开放源代码分布式的跟踪系统，用于收集服务的定时数据，以解决微服务架构中的延迟问题，包括数据的收集、存储、查找和展现。
- Pinpoint   
Pinpoint是一款对Java编写的大规模分布式系统的APM工具，由韩国人开源的分布式跟踪组件。
- Skywalking   
国产的优秀APM组件，是一个对JAVA分布式应用程序集群的业务运行情况进行追踪、告警和分析的系统。

其他类似的组件还有美团点评的CAT，淘宝的鹰眼EgleEye。

如上所述，那么我们选择链路监控组件有哪些要求呢？Dapper中也提到了，笔者总结如下：

- 探针的性能消耗。   
APM组件服务的影响应该做到足够小。在一些高度优化过的服务，即使一点点损耗也会很容易察觉到，而且有可能迫使在线服务的部署团队不得不将跟踪系统关停。

- 代码的侵入性   
对于应用的程序员来说，是不需要知道有跟踪系统这回事的。如果一个跟踪系统想生效，就必须需要依赖应用的开发者主动配合，那么这个跟踪系统也太脆弱了，往往由于跟踪系统在应用中植入代码的bug或疏忽导致应用出问题，这样才是无法满足对跟踪系统“无所不在的部署”这个需求。
- 可扩展性   
能够支持的组件越多当然越好。或者提供便捷的插件开发API，对于一些没有监控到的组件，应用开发者也可以自行扩展。

- 数据的分析   
数据的分析要快 ，分析的维度尽可能多。跟踪系统能提供足够快的信息反馈，就可以对生产环境下的异常状况做出快速反应。分析的全面，能够避免二次开发。


## 2. 基础概念
上面列出的几种组件，其中Zipkin是严格按照`Google Dapper`论文实现的，下面介绍下其中涉及的基本概念。

- Span
基本工作单元，一次链路调用(可以是RPC，DB等没有特定的限制)创建一个span，通过一个64位ID标识它，uuid较为方便，span中还有其他的数据，例如描述信息，时间戳，key-value对的(Annotation)tag信息，parent-id等,其中parent-id可以表示span调用链路来源。

- Trace:类似于树结构的Span集合，表示一条调用链路，存在唯一标识。比如你运行的分布式大数据存储一次Trace就由你的一次请求组成。

- Annotation: 注解,用来记录请求特定事件相关信息(例如时间)，通常包含四个注解信息：   
	(1) cs：Client Start,表示客户端发起请求

   (2) sr：Server Receive,表示服务端收到请求

   (3) ss：Server Send,表示服务端完成处理，并将结果发送给客户端

   (4) cr：Client Received,表示客户端获取到服务端返回信息

### 2.1 Trace

下面看一下，在系统中Trace是什么样子。 

![trace](http://ovcjgn2x0.bkt.clouddn.com/trace-id.png "trace")

每种颜色的note标注了一个span，一条链路通过TraceId唯一标识，Span标识发起的请求信息。树节点是整个架构的基本单元，而每一个节点又是对span的引用。节点之间的连线表示的span和它的父span直接的关系。虽然span在日志文件中只是简单的代表span的开始和结束时间，他们在整个树形结构中却是相对独立的。

### 2.2 Span 
![sp](http://ovcjgn2x0.bkt.clouddn.com/span.png "span")

上图说明了span在一次大的跟踪过程中是什么样的。Dapper记录了span名称，以及每个span的ID和父ID，以重建在一次追踪过程中不同span之间的关系。如果一个span没有父ID被称为root span。所有span都挂在一个特定的跟踪上，也共用一个跟踪id。

### 2.3 Annotation
自动的探针，不需要修改应用程序源代码，对应用开发者近乎零浸入的成本对分布式控制路径进行跟踪，几乎完全依赖于基于少量通用组件库的改造。Dapper还允许应用程序开发人员在Dapper跟踪的过程中添加额外的信息，以监控更高级别的系统行为，或帮助调试问题。

下面章节将会介绍下上述三种APM组件的使用与实践。

## 3. zipkin
zipkin主要涉及几个组件：`collector`收集agent的数据，`storage`存储，`web UI`图形化界面，`search`查询Storage中存储的数据,提供简单的JSON API获取数据。
![](http://ovcjgn2x0.bkt.clouddn.com/architecture-1.png "zipkin组件") 

我们的项目基于微服务框架spring cloud构建微服务。spring cloud也提供了spring-cloud-sleuth来方便集成zipkin实现。所以笔者就在项目中试了下spring-cloud-sleuth-zipkin。

起了三个服务：
zipkin-server、zipkin-client-backend、zipkin-client。   
其中server服务负责收集以及信息展示。client-backend调用client，产生调用链路信息。

### 3.1 zipkin-server实现
zipkin-server实现主要有两点需要注意，其一是收集到数据的存储，方式包括内存、数据库、ES等；其二是通信方式，包括http通信和mq异步方式通信，http通信会对正常的访问造成影响，所以还是推荐基于mq异步方式通信。

本文使用mysql作为存储，使用MQ通信，MQ通信基于Spring-cloud-Stream。本文重点不在zipkin-server的具体几种实现方式，其他方式，读者可以自己去官网查看。

（1）pom需要添加的引用如下：


```xml
	<dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>

    <!--zipkin依赖-->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-sleuth-zipkin-stream</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
    </dependency>
    <dependency>
        <groupId>io.zipkin.java</groupId>
        <artifactId>zipkin-autoconfigure-ui</artifactId>
        <scope>runtime</scope>
    </dependency>

    <!--保存到数据库需要如下依赖-->
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jdbc</artifactId>
    </dependency>

```

（2）启动类：

```java
// 使用Stream方式启动ZipkinServer
@EnableZipkinStreamServer
@SpringBootApplication
public class ZipkinStreamServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ZipkinStreamServerApplication.class,args);
    }
}
```
`@EnableZipkinStreamServer`注解引入了`@EnableZipkinServer`注解，同时还创建了一个rabbit-mq的SleuthSink消息队列监听器。

（3）配置文件

```yml
server:
  port: 9411

spring:
  datasource:
    username: root
    password: root123
    schema[0]: classpath:/zipkin.sql

zipkin:
  storage:
    type: mysql

---
spring:
  application:
    name: microservice-zipkin-stream-server
  rabbitmq:
    host: ${RABBIT_ADDR:localhost}
    port: ${RABBIT_PORT:5672}
    username: guest
    password: guest
  sleuth:
    enabled: false
  profiles: default
  datasource:
    url:  jdbc:mysql://localhost:3307/zipkin?autoReconnect=true&useSSL=false

```

zipkin.sql可以去官网获取，设置了zipkin-server的端口号为9411。

### 3.2 zipkin-client
两个zipkin-client的配置一样，所以放在一起。

（1）pom依赖

```xml
		<dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-sleuth-zipkin-stream</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-sleuth</artifactId>
        </dependency>

```

(2) 配置文件

```yml
spring:
  rabbitmq:
    host: 127.0.0.1
    port : 5672
    username: guest
    password: guest

```

### 3.3 结果

服务之间的调用关系如下：

![](http://ovcjgn2x0.bkt.clouddn.com/zipdependency1.jpg "zipdependency.jpg")

可以看到客户端的请求经过gateway，调用内网中的各个服务，部分还涉及到调用notice服务。从图中可以清楚的看出客户端请求所经过的服务。   
下面看下demo2-default服务实例中的http path：

![](http://ovcjgn2x0.bkt.clouddn.com/zipkintrace.jpg "zipkintrace")

上图中demo2-default服务的几个http path按照时长排序，显示了trace调用时长和span数量。点进去可以看到：

![](http://ovcjgn2x0.bkt.clouddn.com/trace-demo2.jpg "demo2")

图中列出了从父span开始，每一个span的耗时。本次trace中，涉及到两个服务demo1和demo2。demo2调用demo1，从597ms开始调用demo1，完成最终的请求总共耗时1265ms。

## 4. pinpoint
对代码零侵入，运用JavaAgent字节码增强技术，只需要加启动参数即可。
pinpoint的几个组件部分和zipkin差不多，架构图如下：

![](http://ovcjgn2x0.bkt.clouddn.com/pinpoint-architecture.png "pinpoint")

Pinpoint-Collector收集各种性能数据、Pinpoint-Agent和自己运行的应用关联起来的探针、Pinpoint-Web将收集到的数据显示成WEB网页形式、HBase Storage收集到的数据存到HBase中。

### 4.1 pinpoint安装
主要涉及以下软件的安装：

- jdk 1.8  
Java环境必须的，没啥好解释。

- Hbase   
pinpoint收集来的测试数据，主要是存在Hbase数据库的。所以它可以收集大量的数据，可以进行更加详细的分析。Hbase安装完成后，需要初始化Hbase的pinpoint库，由pinpoint提供。Hbase内置了zookeeper。

- pinpoint-collector   
collector收集agent的数据，将数据存到hbase集群，对外暴露collector的tcp和udp的监听端口9994，9995，9996。

- pinpoint-web   
页面展示，配置文件中设置环境变量HBASE_HOST、HBASE_PORT等。

- pinpoint-agent

到[官网](https://github.com/naver/pinpoint/wiki)release页面下载pinpoint-agent-x-SNAPSHOT.tar.gz，配置pinpoint.config中相关collector的信息。

**安装确实还比较麻烦，本文篇幅太长了，具体步骤后面再单独写文章讲解。**

### 4.2 运行pinpoint-agent
笔者使用的是spring-boot项目，所以只需要在启动jar包的命令中加入-javaagent参数，并指定pinpoint-bootstrap包的绝对路径。实例代码如下：

```bash
java -javaagent:/aoho/auth_compose/pinpoint-bootstrap-1.6.0.jar -Dpinpoint.agentId=aoho-consumer -Dpinpoint.applicationName=aoho-consumer -jar id_generator/snowflake-id-generate-1.0-SNAPSHOT.jar
```
起的id生成器服务比较简单，没有用到数据库等存储介质。服务注册到consul上，本地客户端请求了id-server获取id。其调用链如下：

![call tree](http://ovcjgn2x0.bkt.clouddn.com/Jietu20171106-215536.jpg "pinpoint 调用链")

pinpoint提供的功能比较丰富，下图是调用/api/id接口的详细信息。


![api](http://ovcjgn2x0.bkt.clouddn.com/Jietu20171106-215606.jpg "接口信息")

可以看到，pinpoint记录了客户端的相应时间、IP地址等，调用树在下面也有详细列出，每个方法的耗时等。

![serverMap](http://ovcjgn2x0.bkt.clouddn.com/Jietu20171106-215642.jpg "serverMap")

serverMap中还展示了服务器的堆、永久代、CPU等信息，非常强大。

## 5. Skywalking

Skywalking是国内开源的APM监控组件，官网[OpenSkywalking](https://github.com/OpenSkywalking/)，根据官网介绍，其着力于性能和实时性两方面。   
网上找到的Skywalking的架构图。

![](http://ovcjgn2x0.bkt.clouddn.com/skyarche.png "")

可以看到Skywalking也是四部分组成，collector、agent、web、storage。支持集群部署，集群之间还引入了grpc通信。存储支持内置的h2和elasticsearch存储。

### 5.1 安装
具体安装可见[官网](https://github.com/OpenSkywalking/skywalking/wiki/3.2.3-%E9%9B%86%E7%BE%A4%E6%A8%A1%E5%BC%8F%E9%83%A8%E7%BD%B2Collector)。

- collector安装   
此处笔者使用单机版的collector，在release页面下载好压缩包，解压后，单机版的collector默认使用h2数据库，所以配置文件可以不需要修改，即可以运行`bin/startup.sh`。 

![](http://ovcjgn2x0.bkt.clouddn.com/collectorsky.jpg "")

目录结构如上，logs文件夹中，有启动的日志，可以查看启动情况。

- web   
解压好skywalking-ui，设置server的config/collector_config.properties、log4j2以及监听端口等相关信息，

- agent   
拷贝skywalking-agent目录到所需位置，探针包含整个目录，设置/config/agent.config中的collector信息。

### 5.2 运行agent
Spring boot的项目，启动和上面pinpoint agent启动方式相同。增加JVM启动参数，`-javaagent:/path/to/skywalking-agent/skywalking-agent.jar`。

这次起了user服务，涉及到mysql、redis、consul等组件。可以看到其调用链路图如下：

![](http://ovcjgn2x0.bkt.clouddn.com/skydependency.jpg "")

当访问`/api/external/register-code`和`/api/external/validate-code`接口时，形成了上图中的调用链。

![](http://ovcjgn2x0.bkt.clouddn.com/skytree1.jpg "")

上图TraceId为	2.59.15103021777910001的请求`/api/external/register-code`。这次trace中，每个涉及的span的耗时都有在图中统计。

![](http://ovcjgn2x0.bkt.clouddn.com/serversky.jpg "")

上面这张图，是对userService中的Entry Service List接口进行了统计，包括调用数、成功率等信息。（因为内置的h2，这边在UI响应很久）

还有个对instance的统计，统计jvm的相关信息，API响应也很慢，可能与我用的存储有关吧，就不截图了。

## 6. 总结
本文主要写了链路监控组件的实践。首先介绍了链路监控组件产生与应用的背景，以及选择的要求；其次介绍了opentracing中涉及的基本概念；最后大篇幅介绍了三种APM组件的安装与使用，并展示了每种APM的UI截图。本文比较简单，下一篇文章主要介绍几种APM选型的比较与性能测试。


**zipkin-server-stream的源码   
github:  https://github.com/keets2012/Spring-Boot-Samples/   
oschina:  https://gitee.com/keets/spring-boot-samples**


---
### 参考(疯狂找资料)

1. [OpenTracing官方标准-中文版](https://github.com/opentracing-contrib/opentracing-specification-zh)
2. [Skywalking](https://github.com/wu-sheng/sky-walking)
3. [Zipkin](http://zipkin.io/)
4. [PinPoint](https://github.com/naver/pinpoint)
5. [Spring Cloud Sleuth](http://cloud.spring.io/spring-cloud-static/spring-cloud-sleuth/1.2.5.RELEASE/single/spring-cloud-sleuth.html#)
6. [Dapper](http://bigbully.github.io/Dapper-translation/)
7. [pinpoint 安装部署](http://www.cnblogs.com/yyhh/p/6106472.html)
8. [java开源APM概要](https://segmentfault.com/a/1190000006817114)
9. [分布式系统监控系统zipkin入门](http://blog.csdn.net/john1337/article/details/71127056)
10. [跟着小程学微服务-自己动手扩展分布式调用链](http://www.jianshu.com/p/17a015848dc1)



