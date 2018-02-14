---
title: Spring Cloud Bus中的事件的订阅与发布（一）
date: 2018-2-13
categories: 微服务
tags:
- Spring Cloud
---
**年前最后一篇博客更新，提前祝大家新年快乐（还有情人节）！**

下面进入正题。Spring Cloud Bus用轻量级的消息代理将分布式系统的节点连接起来。这可以用来广播状态的该表（比如配置的改变）或者其他关联的指令。一个关键的想法是，总线就像是一个分布式Actuator，用于Spring Boot应用程序的扩展，但它也可以用作应用程序之间的通信通道。Spring Cloud提供了AMQP 传输的代理和Kafka启动Starters，对具有相同的基本功能集的其他传输组件的支持，也在未来的规划中。

## Spring Cloud Bus
`Spring Cloud Bus`是在`Spring Cloud Stream`的基础上进行的封装，对于指定主题的消息的发布与订阅是通过`Spring Cloud Stream`的具体binder实现。因此引入的依赖可以是`spring-cloud-starter-bus-amqp`和`spring-cloud-starter-bus-kafka`其中的一种，分别对应于binder的两种实现。根据上一节的基础应用，我们总结出`Spring Cloud Bus`的主要功能如下两点：

- 对指定主题`springCloudBus`的消息订阅与发布。
- 事件监听，包括刷新事件、环境变更事件、远端应用的ack事件以及本地服务端发送事件等。

下面我们以这两方面作为主线，进行`Spring Cloud Bus`的源码分析。本文主要针对事件的订阅户发布。

## 事件的订阅与发布
### 事件驱动模型
这部分需要读者首先了解下Spring的事件驱动模型。我们在这边简单介绍下设计的主要概念，帮助大家易于理解后面的内容。   
![event-source](http://ovcjgn2x0.bkt.clouddn.com/event-source.jpg "事件驱动模型")

Spring的事件驱动模型由三部分组成：

- 事件：ApplicationEvent，继承自JDK的EventObject，所有事件将继承它，并通过source得到事件源。
- 事件发布者：ApplicationEventPublisher及ApplicationEventMulticaster接口，使用这个接口，我们的Service就拥有了发布事件的能力。
- 事件订阅者：ApplicationListener，继承自JDK的EventListener，所有监听器将继承它。

### 事件的定义
Spring的事件驱动模型的事件定义均继承自`ApplicationEvent`，`Spring Cloud Bus`中有多个事件类，这些事件类都继承了一个重要的抽象类`RemoteApplicationEvent`，我们看一下事件类的类图：

![bus-event](http://ovcjgn2x0.bkt.clouddn.com/bus-event.png "各种事件的定义")

涉及的事件类有：代表了对特定事件确认的事件`AckRemoteApplicationEvent`、环境变更的事件`EnvironmentChangeRemoteApplicationEvent`、刷新事件`RefreshRemoteApplicationEvent`、发送事件 `SentApplicationEvent`、以及未知事件`UnknownRemoteApplicationEvent`。下面我们分别看一下这些事件的定义。

#### 抽象基类：RemoteApplicationEvent   
通过上面的类图，我们知道`RemoteApplicationEvent`是其他事件类的基类，定义了事件对象的公共属性。

```java
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "type") //序列化时使用子类的名称作为type
@JsonIgnoreProperties("source") //序列化时，忽略 source
public abstract class RemoteApplicationEvent extends ApplicationEvent {
	private static final Object TRANSIENT_SOURCE = new Object();
	private final String originService;
	private final String destinationService;
	private final String id;

	protected RemoteApplicationEvent(Object source, String originService,
			String destinationService) {
		super(source);
		this.originService = originService;
		if (destinationService == null) {
			destinationService = "**";
		}

		if (!"**".equals(destinationService)) {
			if (StringUtils.countOccurrencesOf(destinationService, ":") <= 1
					&& !StringUtils.endsWithIgnoreCase(destinationService, ":**")) {
				//destination的所有实例
				destinationService = destinationService + ":**";
			}
		}
		this.destinationService = destinationService;
		this.id = UUID.randomUUID().toString();
	}
	...
}
```
在`RemoteApplicationEvent`中定义了主要的三个通用属性事件的来源originService、事件的目的服务destinationService和随机生成的全局id。通过其构造方法可知，destinationService可以使用通配符的形式{serviceId}:{appContextId}，两个变量都省略的话，则通知到所有服务的所有实例。只省略appContextId时，则对应的destinationService为相应serviceId的所有实例。另外，注解`@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "type")`对应于序列化时，使用子类的名称作为type；而`@JsonIgnoreProperties("source")`表示序列化时，忽略source属性，source定义在JDK中的`EventObject`。

#### EnvironmentChangeRemoteApplicationEvent   
用于动态更新服务实例的环境属性，我们在基础应用中更新`cloud.version`属性时，关联到该事件。

```java
public class EnvironmentChangeRemoteApplicationEvent extends RemoteApplicationEvent {

	private final Map<String, String> values;

	public EnvironmentChangeRemoteApplicationEvent(Object source, String originService,
			String destinationService, Map<String, String> values) {
		super(source, originService, destinationService);
		this.values = values;
	}
	...
}
```
可以看到，`EnvironmentChangeRemoteApplicationEvent`事件类的实现很简单。定义了Map类型的成员变量，key对应于环境变量名，而value对应更新后的值。

#### RefreshRemoteApplicationEvent   
刷新远端应用配置的事件，用于接收远端刷新的请求。

```java
public class RefreshRemoteApplicationEvent extends RemoteApplicationEvent {
	public RefreshRemoteApplicationEvent(Object source, String originService,
			String destinationService) {
		super(source, originService, destinationService);
	}
}
```

继承自抽象事件类`RemoteApplicationEvent`，没有特别的成员属性。

#### AckRemoteApplicationEvent   
确认远端应用事件，该事件表示一个特定的`RemoteApplicationEvent`事件被确认。

```java
public class AckRemoteApplicationEvent extends RemoteApplicationEvent {

	private final String ackId;
	private final String ackDestinationService;
	private Class<? extends RemoteApplicationEvent> event;

	public AckRemoteApplicationEvent(Object source, String originService,
			String destinationService, String ackDestinationService, String ackId,
			Class<? extends RemoteApplicationEvent> type) {
		super(source, originService, destinationService);
		this.ackDestinationService = ackDestinationService;
		this.ackId = ackId;
		this.event = type;
	}
	...
	
	public void setEventName(String eventName) {
		try {
			event = (Class<? extends RemoteApplicationEvent>) Class.forName(eventName);
		} catch (ClassNotFoundException e) {
			event = UnknownRemoteApplicationEvent.class;
		}
	}
}
```
该事件类在`RemoteApplicationEvent`基础上，定义了成员属性ackId、ackDestinationService和event。
ackId和ackDestinationService，分别表示确认的时间的id和对应的目标服务。event对应事件类型，确认事件能够确认的必然是`RemoteApplicationEvent`的子类，因此event属性设值时需要进行检查，如果转换出现异常，则定义为未知的事件类型。这些事件可以被任何需要统计总线事件响应的应用程序来监听。 它们的行为与普通的远程应用程序事件相似，即如果目标服务与本地服务ID匹配，则应用程序会在其上下文中触发该事件。

#### SentApplicationEvent   
发送应用事件，表示系统中的某个地方发送了一个远端事件。

```java
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "type")
@JsonIgnoreProperties("source")
public class SentApplicationEvent extends ApplicationEvent {

	private static final Object TRANSIENT_SOURCE = new Object();
	private final String originService;
	private final String destinationService;
	private final String id;
	private Class<? extends RemoteApplicationEvent> type;

	protected SentApplicationEvent() {
		// for serialization libs like jackson
		this(TRANSIENT_SOURCE, null, null, null, RemoteApplicationEvent.class);
	}

	public SentApplicationEvent(Object source, String originService,
			String destinationService, String id,
			Class<? extends RemoteApplicationEvent> type) {
		super(source);
		this.originService = originService;
		this.type = type;
		if (destinationService == null) {
			destinationService = "*";
		}
		if (!destinationService.contains(":")) {
			// All instances of the destination unless specifically requested
			destinationService = destinationService + ":**";
		}
		this.destinationService = destinationService;
		this.id = id;
	}
	...
}
```
可以看到该事件类继承自`ApplicationEvent`，它本身并不是一个`RemoteApplicationEvent`事件，所以不会通过总线发送，而是在本地生成（多为响应远端事件）。想要审计远端事件的应用可以监听该事件，并且所有的`AckRemoteApplicationEvent`事件中的id来源于相应的`SentApplicationEvent`中定义的id。在其定义的成员属性中，相比于远端应用事件多了一个事件类型type，该类型限定于`RemoteApplicationEvent`的子类。

#### UnknownRemoteApplicationEvent   
未知的远端应用事件，也是`RemoteApplicationEvent`事件类的子类。该事件类与之前的`SentApplicationEvent`、`AckRemoteApplicationEvent`有关，当序列化时遇到事件的类型转换异常，则自动构造成一个未知的远端应用事件。

**事件监听器以及消息的订阅与发布待后续更新。。**

### 参考
[Spring Cloud Bus-v1.3.3](http://cloud.spring.io/spring-cloud-static/spring-cloud-bus/1.3.3.RELEASE/single/spring-cloud-bus.html)

