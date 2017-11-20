---
title: 微服务之分布式配置中心Cloud Config
date: 2017-11-20
categories: 微服务
tags:
- Spring Cloud
---
# 微服务之分布式配置中心Cloud Config

## 1. 分布式配置中心

分布式系统中，服务数量剧增，其配置文件需要实现统一管理并且能够实时更新，分布式配置中心组件必然是需要的。Spring Cloud提供了配置中心组件Spring Cloud Config ，它支持配置服务放在远程Git仓库和本地文件中。默认采用git来存储配置信息，笔者示例也是采用默认的git repository，这样通过git客户端工具来方便的管理和访问配置内容。

![配置服务器工作示意图](http://ovcjgn2x0.bkt.clouddn.com/%E9%85%8D%E7%BD%AE%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%B7%A5%E4%BD%9C%E7%A4%BA%E6%84%8F%E5%9B%BE.jpg "配置服务器工作示意图")

在Spring Cloud Config 组件中，有两个角色，一是Config Server配置服务器，为其他服务提供配置文件信息；另一个是Config Client即其他服务，启动时从Config Server拉取配置。   
下面分别介绍下Config Server和Config Client的搭建实现方法。

## 2. 配置服务器Config Server
### 2.1 pom中的jars
只需要添加如下两个jar包的引用。

```xml
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-server</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-consul-discovery</artifactId>
            <exclusions>
                <exclusion>
                    <artifactId>jsr311-api</artifactId>
                    <groupId>javax.ws.rs</groupId>
                </exclusion>
            </exclusions>
        </dependency>
    </dependencies>
```

因为Spring Cloud Config服务器为客户端提供配置是要通过服务发现，所以这边引入consul的starter，配置服务器和客户端都注册到consul集群中。

### 2.2 入口类
简单，因为Spring Cloud 提供了很多开箱即用的功能，通过spring-cloud-config-server的注解激活配置服务器。

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```
`@EnableConfigServer`注解很重要，将该服务标注为配置服务器；`@EnableDiscoveryClient`注册服务，供其他服务发现调用。

### 2.3 bootstrap.yml

```yml
server:
  port: 8888

spring:
  application:
    name: config-server
  cloud:
      consul:
        discovery:
          preferIpAddress: true
          enabled: true
          register: true
          service-name: config-service
          //...
        host: localhost
        port: 8500
---
spring:
  cloud:
    config:
      server:
        git:
          uri: https://gitee.com/keets/Config-Repo.git
          searchPaths: ${APP_LOCATE:dev}
          username: user
          password: pwd
```

配置第一段指定了服务的端口；第二段是服务发现相关的配置；第三段是配置服务器的信息，这里将配置文件存储在码云上，默认的搜索文件路径为dev文件夹，可以通过环境变量指定，再下面是用户名和密码，公开的项目不需要设置用户名和密码。

至此配置服务器已经搭建完成，是不是很简单？

## 3. 配置的git仓库
配置服务器配置的仓库是`https://gitee.com/keets/Config-Repo.git`。笔者在这个仓库中建了两个文件夹：dev和prod。并且在dev文件夹中新建了文件configclient-dev.yml。为什么这样命名，能随便命名吗？答案是不可以，下面我们看下config文件的命名规则。

URL与配置文件的映射关系如下：

- /{application}/{profile}[/{label}]
- /{application}-{profile}.yml
- /{label}/{application}-{profile}.yml
- /{application}-{profile}.properties
- /{label}/{application}-{profile}.properties
上面的url会映射{application}-{profile}.yml对应的配置文件，{label}对应git上不同的分支，默认为master
比如Config Client，{application}对应`spring.application：configclient`，exp对应{profile}，{label}不指定则默认为master。

新建的configclient-dev.yml如下：

```yml
spring:
  profiles: dev

cloud:
  version: Dalston.SR4
```
## 4. 配置客户端Config Client
### 4.1 pom中的jars
只需要添加如下两个jar包的引用。

```xml
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-server</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-consul-discovery</artifactId>
            <exclusions>
                <exclusion>
                    <artifactId>jsr311-api</artifactId>
                    <groupId>javax.ws.rs</groupId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
    		<groupId>org.springframework.boot</groupId>
    		<artifactId>spring-boot-starter-actuator</artifactId>
		</dependency>
    </dependencies>
```

新增spring-boot-starter-actuator监控模块，为了配置信息是动态刷新，其中包含了/refresh刷新API。其他和配置服务器中添加的相同，没啥可说。

### 4.2 入口类
简单，因为Spring Cloud 提供了很多开箱即用的功能，通过spring-cloud-config-server的注解激活配置服务器。

```java
@SpringBootApplication
@EnableDiscoveryClient
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```
配置客户端不需要`@EnableConfigServer`注解。

### 4.3 bootstrap.yml

```yml

server:
  port: 9901
cloud:
  version: Brixton SR7

spring:
  cloud:
    consul:
      discovery:
        preferIpAddress: true
        enabled: true
        register: true
        service-name: config-client
      //...
      host: localhost
      port: 8500

---
spring:
  profiles:
    active: dev
  application:
  	#app名称
    name: configclient
  cloud:
    config:
      #指定profile
      profile: dev
      label: master
      discovery:
        enabled: true
        #server名
        service-id: config-service
      enabled: true
      fail-fast: true

---
spring:
  profiles: default
  application:
    name: configclient 

```

从上面配置可以看出我们所激活的profile是dev，配置的配置服务名为`config-service`，指定从master分支拉取配置。所以configclient启动时会去配置服务器中拉取对应的configclient-dev的配置文件信息。

### 4.4 TestResource
笔者新建了一个TestResource，对应的API端点为/api/test。

```java
    @Value("${cloud.version}")
    private String version;

    @GetMapping("/test")
    public String from() {
        return version;
    }
```
`cloud.version`可以在上面的配置文件看到默认指定的是Brixton SR7，而笔者在配置中心设置的值为Dalston.SR4。

## 5. 测试结果
### 5.1 获取配置
首先看一下配置客户端启动时的日志信息，是不是真的按照我们配置的，从配置服务器拉取configclient-dev信息。

![ccstart](http://ovcjgn2x0.bkt.clouddn.com/configstartupload.jpg "Config Client启动")

从日志看来，是符合的上面的配置猜想的。我们再从配置客户端提供的API接口进一步验证。

![pj](http://ovcjgn2x0.bkt.clouddn.com/realconfig.jpg "配置结果")

可以看到确实是Dalston.SR4，配置服务能够正常运行。

### 5.2 动态刷新配置
Spring Cloud Config还可以实现动态更新配置的功能。下面我们修改下Config Repo中的cloud.version配置为Camden SR7，并通过刷新config client的/refresh端点来应用配置。   
从下图可以看到结果是预期所想。

![rr](http://ovcjgn2x0.bkt.clouddn.com/refrestresult.jpg "刷新配置")

这边配置的刷新是通过手工完成了，还可以利用githook进行触发。当本地提交代码到git后，调用了下图设置的url。有两个端点可以使用：

- refresh：以post方式执行/refresh 会刷新env中的配置
- restart：如果配置信息已经注入到bean中，由于bean是单例的，不会去加载修改后的配置 
需要通过post方式去执行/restart, 还需要配置endpoints.restart.enabled: true。

第二种情况比较耗时，`@RefreshScope`是spring cloud提供的注解，在执行refresh时会刷新bean中变量值。
> Convenience annotation to put a @Bean definition in RefreshScope.
 Beans annotated this way can be refreshed at runtime and any components that are using them will get a new instance on the next method call, fully initialized and injected with all dependencies.

上面说了RefreshScope是一个方便的bean注解，加上这个注解可以在运行态刷新bean。其他使用该bean的components下次调用时会获取一个已经被初始化好和注入的新实例对象。

githook设置页面如下。

![githook](http://ovcjgn2x0.bkt.clouddn.com/webhooks.jpg "githook设置")

## 6. 总结
本文主要讲了配置服务器和配置客户端的搭建过程，最后通过配置客户端的日志和端点信息验证是否能成功使用配置服务中心。总体来说，非常简单。文中的部分配置没有写完整，读者需要可以看文末的git项目。   

不过关于配置中心，本文的讲解并不完整，下一篇文章将会讲解配置服务器与消息总线的结合使用，实现对配置客户端的自动更新及灰度发布等功能。

**本文源码
github: https://github.com/keets2012/Spring-Boot-Samples/   
gitee: https://gitee.com/keets/spring-boot-samples/**

---
### 参考
1. [Spring Cloud Config](http://cloud.spring.io/spring-cloud-static/Dalston.SR4/single/spring-cloud.html)
2. [分布式配置中心](http://m.blog.csdn.net/forezp/article/details/70037291)
