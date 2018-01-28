---
title: Spring Cloud 覆写远端的配置属性
date: 2018-1-23
categories: 微服务
tags:
- Spring Cloud
---
## 覆写远端的配置属性

应用的配置源通常都是远端的Config Server服务器，默认情况下，本地的配置优先级低于远端配置仓库。如果想实现本地应用的系统变量和config文件覆盖远端仓库中的属性值，可以通过如下设置：

```yaml
spring:
  cloud:
    config:
      allowOverride: true
      overrideNone: true
      overrideSystemProperties: false
```

- overrideNone：当allowOverride为true时，overrideNone设置为true，外部的配置优先级更低，而且不能覆盖任何存在的属性源。默认为false
- allowOverride：标识overrideSystemProperties属性是否启用。默认为true，设置为false意为禁止用户的设置
- overrideSystemProperties：用来标识外部配置是否能够覆盖系统属性，默认为true

客户端通过如上配置，可以实现本地配置优先级更高，且不能被覆盖。由于我们基于的Spring Cloud当前版本是`Edgware.RELEASE`，上面的设置并不能起作用，而是使用了`PropertySourceBootstrapProperties`中的默认值。具体情况见issue：`https://github.com/spring-cloud/spring-cloud-commons/pull/250`，我们在下面分析时会讲到具体的bug源。


## 源码分析
### ConfigServicePropertySourceLocator
覆写远端的配置属性归根结底与客户端的启动时获取配置有关，在获取到配置之后如何处理？我们看一下spring cloud config中的资源获取类`ConfigServicePropertySourceLocator`的类图。

![ConfigServicePropertySourceLocator](http://ovcjgn2x0.bkt.clouddn.com/locator.png "ConfigServicePropertySourceLocator类图")

`ConfigServicePropertySourceLocator`实质是一个属性资源定位器，其主要方法是`locate(Environment environment)`。首先用当前运行应用的环境的application、profile和label替换configClientProperties中的占位符并初始化RestTemplate，然后遍历labels数组直到获取到有效的配置信息，最后还会根据是否快速失败进行重试。主要流程如下：

![locate](http://ovcjgn2x0.bkt.clouddn.com/locatorconfig.jpg "locate处理流程")

`locate(Environment environment)`调用`getRemoteEnvironment(restTemplate, properties, label, state)`方法通过http的方式获取远程服务器上的配置数据。实现也很简单，显示替换请求路径path中占位符，然后进行头部headers组装，组装好了就可以发送请求，最后返回结果。	
在上面的实现中，我们看到获取到的配置信息存放在`CompositePropertySource`，那是如何使用它的呢？这边补充另一个重要的类是PropertySourceBootstrapConfiguration，它实现了ApplicationContextInitializer接口，该接口会在应用上下文刷新之前`refresh()`被回调，从而执行初始化操作，应用启动后的调用栈如下：

```
SpringApplicationBuilder.run() -> SpringApplication.run() -> SpringApplication.createAndRefreshContext() -> SpringApplication.applyInitializers() -> PropertySourceBootstrapConfiguration.initialize()
```

### PropertySourceBootstrapConfiguration
而上述`ConfigServicePropertySourceLocator`的locate方法会在initialize中被调用，从而保证上下文在刷新之前能够拿到必要的配置信息。具体看一下initialize方法：

```java
public class PropertySourceBootstrapConfiguration implements
		ApplicationContextInitializer<ConfigurableApplicationContext>, Ordered {

	private int order = Ordered.HIGHEST_PRECEDENCE + 10;

	@Autowired(required = false)
	private List<PropertySourceLocator> propertySourceLocators = new ArrayList<>();

	@Override
	public void initialize(ConfigurableApplicationContext applicationContext) {
		CompositePropertySource composite = new CompositePropertySource(
				BOOTSTRAP_PROPERTY_SOURCE_NAME);
		//对propertySourceLocators数组进行排序，根据默认的AnnotationAwareOrderComparator
		AnnotationAwareOrderComparator.sort(this.propertySourceLocators);
		boolean empty = true;
		//获取运行的环境上下文
		ConfigurableEnvironment environment = applicationContext.getEnvironment();
		for (PropertySourceLocator locator : this.propertySourceLocators) {
			//遍历this.propertySourceLocators
			PropertySource<?> source = null;
			source = locator.locate(environment);
			if (source == null) {
				continue;
			}
			logger.info("Located property source: " + source);
			//将source添加到PropertySource的链表中
			composite.addPropertySource(source);
			empty = false;
		}
		//只有source不为空的情况，才会设置到environment中
		if (!empty) {
			//返回Environment的可变形式，可进行的操作如addFirst、addLast
			MutablePropertySources propertySources = environment.getPropertySources();
			String logConfig = environment.resolvePlaceholders("${logging.config:}");
			LogFile logFile = LogFile.get(environment);
			if (propertySources.contains(BOOTSTRAP_PROPERTY_SOURCE_NAME)) {
				//移除bootstrapProperties
				propertySources.remove(BOOTSTRAP_PROPERTY_SOURCE_NAME);
			}
			//根据config server覆写的规则，设置propertySources
			insertPropertySources(propertySources, composite);
			reinitializeLoggingSystem(environment, logConfig, logFile);
			setLogLevels(environment);
			//处理多个active profiles的配置信息
			handleIncludedProfiles(environment);
		}
	}
	//...
}
```


下面我们看一下，在`initialize`方法中进行了哪些操作。

- 根据默认的 AnnotationAwareOrderComparator 排序规则对propertySourceLocators数组进行排序
- 获取运行的环境上下文ConfigurableEnvironment
- 遍历propertySourceLocators时
	- 调用 locate 方法，传入获取的上下文environment
	- 将source添加到PropertySource的链表中
	- 设置source是否为空的标识标量empty
- source不为空的情况，才会设置到environment中
	- 返回Environment的可变形式，可进行的操作如addFirst、addLast
	- 移除propertySources中的bootstrapProperties
	- 根据config server覆写的规则，设置propertySources
	- 处理多个active profiles的配置信息

初始化方法`initialize`处理时，先将所有PropertySourceLocator类型的对象的`locate`方法遍历，然后将各种方式得到的属性值放到CompositePropertySource中，最后调用`insertPropertySources(propertySources, composite)`方法设置到Environment中。Spring Cloud Context中提供了覆写远端属性的`PropertySourceBootstrapProperties`，利用该配置类进行判断属性源的优先级。

```java
	private void insertPropertySources(MutablePropertySources propertySources,
			CompositePropertySource composite) {
		MutablePropertySources incoming = new MutablePropertySources();
		incoming.addFirst(composite);
		PropertySourceBootstrapProperties remoteProperties = new PropertySourceBootstrapProperties();
		new RelaxedDataBinder(remoteProperties, "spring.cloud.config")
				.bind(new PropertySourcesPropertyValues(incoming));
		//如果不允许本地覆写
		if (!remoteProperties.isAllowOverride() || (!remoteProperties.isOverrideNone()
				&& remoteProperties.isOverrideSystemProperties())) {
			propertySources.addFirst(composite);
			return;
		}
		//overrideNone为true，外部配置优先级最低
		if (remoteProperties.isOverrideNone()) {
			propertySources.addLast(composite);
			return;
		}
		if (propertySources
				.contains(StandardEnvironment.SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME)) {
			//根据overrideSystemProperties，设置外部配置的优先级
			if (!remoteProperties.isOverrideSystemProperties()) {
				propertySources.addAfter(
						StandardEnvironment.SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME,
						composite);
			}
			else {
				propertySources.addBefore(
						StandardEnvironment.SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME,
						composite);
			}
		}
		else {
			propertySources.addLast(composite);
		}
	}
```
上述实现主要是根据`PropertySourceBootstrapProperties`中的属性，调整多个配置源的优先级。从其实现可以看到 `PropertySourceBootstrapProperties` 对象的是被直接初始化，使用的是默认的属性值而并未注入我们在配置文件中设置的。

修复后的实现：

```java
@Autowired(required = false)
private PropertySourceBootstrapProperties remotePropertiesForOverriding;
private void insertPropertySources(MutablePropertySources propertySources, CompositePropertySource composite) {
        MutablePropertySources incoming = new MutablePropertySources();
  	incoming.addFirst(composite);
 	PropertySourceBootstrapProperties remoteProperties = remotePropertiesForOverriding == null ? new PropertySourceBootstrapProperties() : remotePropertiesForOverriding;
        ...
    
}
```

### 参考
[Spring Cloud Edgware.RELEASE](http://cloud.spring.io/spring-cloud-static/Edgware.RELEASE/single/spring-cloud.html)

