---
title: Spring中的事件驱动模型（一）
date: 2018-2-22 
categories: java
tags:
- Spring
- Java
---
正月初七，新年第一篇。

## 事件驱动模型简介
事件驱动模型通常也被理解成观察者或者发布/订阅模型。

- 是一种对象间的一对多的关系；
- 当目标发送改变（发布），观察者（订阅者）就可以接收到改变；
- 观察者如何处理，目标无需干涉，它们之间的关系是松耦合的。

![event-source](http://ovcjgn2x0.bkt.clouddn.com/event-source.jpg "事件驱动模型")

事件驱动模型的例子很多，如生活中的红绿灯，以及我们在微服务中用到的配置中心，当有配置提交时出发具体的应用实例更新Spring上下文环境。

## Spring的事件机制

### 基本概念
Spring的事件驱动模型由三部分组成：

- 事件：ApplicationEvent，继承自JDK的EventObject，所有事件将继承它，并通过source得到事件源。
- 事件发布者：ApplicationEventPublisher及ApplicationEventMulticaster接口，使用这个接口，我们的Service就拥有了发布事件的能力。
- 事件订阅者：ApplicationListener，继承自JDK的EventListener，所有监听器将继承它。

### Spring事件驱动过程

#### 事件
Spring 默认对 ApplicationEvent 事件提供了如下实现：

- ContextStoppedEvent：ApplicationContext停止后触发的事件；
- ContextRefreshedEvent：ApplicationContext初始化或刷新完成后触发的事件；
- ContextClosedEvent：ApplicationContext关闭后触发的事件。如web容器关闭时自动会触发Spring容器的关闭，如果是普通java应用，需要调用`ctx.registerShutdownHook()`注册虚拟机关闭时的钩子才行；
- ContextStartedEvent：ApplicationContext启动后触发的事件；

![eventobject](http://ovcjgn2x0.bkt.clouddn.com/eventobject.png "事件")


```java
public abstract class ApplicationEvent extends EventObject {
    private static final long serialVersionUID = 7099057708183571937L;
    //事件发生的时间
    private final long timestamp = System.currentTimeMillis();
	//创建一个新的ApplicationEvent事件
    public ApplicationEvent(Object source) {
        super(source);
    }

    public final long getTimestamp() {
        return this.timestamp;
    }
}
```
事件基类`ApplicationEvent`，所有的具体事件都会继承该抽象事件类。
#### 事件监听者

![ApplicationListener](http://ovcjgn2x0.bkt.clouddn.com/ApplicationListener.png "事件监听")
`ApplicationListener`继承自JDK的`EventListener`，JDK要求所有监听器将继承它。

```java
@FunctionalInterface
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {
    void onApplicationEvent(E var1);
}
```
提供了onApplicationEvent方法，用以处理`ApplicationEvent`，不过对于具体事件的处理需要进行判断。而`GenericApplicationListener`和`SmartApplicationListener`提供了关于事件更多的元数据信息。

```java
public class SourceFilteringListener implements GenericApplicationListener, SmartApplicationListener {

	private final Object source;

	@Nullable
	private GenericApplicationListener delegate;

	//为特定事件源创建SourceFilteringListener，并传入代理的监听器类
	public SourceFilteringListener(Object source, ApplicationListener<?> delegate) {
		this.source = source;
		this.delegate = (delegate instanceof GenericApplicationListener ?
				(GenericApplicationListener) delegate : new GenericApplicationListenerAdapter(delegate));
	}
	//....省略部分代码
	
	@Override
	public int getOrder() {
		return (this.delegate != null ? this.delegate.getOrder() : Ordered.LOWEST_PRECEDENCE);
	}

	//过滤之后实际处理事件
	protected void onApplicationEventInternal(ApplicationEvent event) {
		//...
		this.delegate.onApplicationEvent(event);
	}

}
```

`SourceFilteringListener`是`ApplicationListener`的装饰器类，过滤特定的事件源。只会注入其事件对应的代理监听器，还提供了按照顺序触发监听器等功能。
在启动的时候会加载一部分 `ApplicationListener`。Spring Context加载初始化完成（refresh）后会再次检测应用中的 `ApplicationListener`，并且注册，此时会将我们实现的 `ApplicationListener` 就会加入到 `SimpleApplicationEventMulticaster` 维护的 Listener 集合中。
Spring也支持直接注解的形式进行事件监听`@EventListener(Event.class)`。

#### 事件发布

![EventPublisher](http://ovcjgn2x0.bkt.clouddn.com/EventPublisher.png "发布事件")
`ApplicationContext`接口继承了`ApplicationEventPublisher`，并在`AbstractApplicationContext`实现了具体代码，实际执行是委托给`ApplicationEventMulticaster`。

```java
@FunctionalInterface
public interface ApplicationEventPublisher {
	//通知所有的注册该事件的应用，事件可以是框架事件如RequestHandledEvent或者特定的应用事件。
    default void publishEvent(ApplicationEvent event) {
        this.publishEvent((Object)event);
    }

    void publishEvent(Object var1);
}
```
实际的执行是委托给，读者有兴趣可以看一下`AbstractApplicationContext`中这部分的逻辑。下面我们具体看一下`ApplicationEventMulticaster`接口中定义的方法。

```java
public interface ApplicationEventMulticaster {

	//增加监听者
	void addApplicationListener(ApplicationListener<?> listener);
	//...

	//移除监听者
	void removeApplicationListener(ApplicationListener<?> listener);
	//...
	
	//广播特定事件给监听者
	void multicastEvent(ApplicationEvent event, @Nullable ResolvableType eventType);

}
```
`AbstractApplicationContext`中定义了对监听者的操作维护，如增加和删除，并提供了将特定事件进行广播的方法。下面看一下具体实现类`SimpleApplicationEventMulticaster`。`ApplicationContext`自动到本地容器里找一个`ApplicationEventMulticaster`实现，如果没有则会使用默认的`SimpleApplicationEventMulticaster`。

```java
public class SimpleApplicationEventMulticaster extends AbstractApplicationEventMulticaster {

	@Nullable
	private Executor taskExecutor;

	//...
	
	//用给定的beanFactory创建SimpleApplicationEventMulticaster
	public SimpleApplicationEventMulticaster(BeanFactory beanFactory) {
		setBeanFactory(beanFactory);
	}

	@Override
	public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
		ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
		for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
			Executor executor = getTaskExecutor();
			if (executor != null) {
				executor.execute(() -> invokeListener(listener, event));
			}
			else {
				invokeListener(listener, event);
			}
		}
	}

	//注入给定事件的给定监听器
	protected void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {
		ErrorHandler errorHandler = getErrorHandler();
		if (errorHandler != null) {
			try {
				doInvokeListener(listener, event);
			}
			...
		}
		else {
			doInvokeListener(listener, event);
		}
	}

	@SuppressWarnings({"unchecked", "rawtypes"})
	private void doInvokeListener(ApplicationListener listener, ApplicationEvent event) {
		try {
			listener.onApplicationEvent(event);
		}
		//...
	}

}
```
在`multicastEvent`方法中，executor不为空的情况下，可以看到是支持异步发布事件。发布事件时只需要调用`ApplicationContext`中的`publishEvent`方法即可进行事件的发布。

## 总结
本文主要介绍了Spring中的事件驱动模型相关概念。首先介绍事件驱动模型，也可以说是观察者模式，在我们的日常生活中和应用开发中有很多应用。随后重点篇幅介绍了Spring的事件机制，Spring的事件驱动模型由事件、发布者和订阅者三部分组成，结合Spring的源码分析了这三部分的定义与实现。笔者将会在下一篇文章，结合具体例子以及Spring Cloud Config中的实现进行实战讲解。

### 参考
1. [事件驱动模型简介](http://jinnianshilongnian.iteye.com/blog/1902886)
2. [Spring事件驱动模型与观察者模式](http://zhangh.tk/2017/08/14/Spring%E4%BA%8B%E4%BB%B6%E9%A9%B1%E5%8A%A8%E6%A8%A1%E5%9E%8B%E4%B8%8E%E8%A7%82%E5%AF%9F%E8%80%85%E6%A8%A1%E5%BC%8F/)
