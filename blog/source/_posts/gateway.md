---
title: 微服务网关netflix-zuul
date: 2017-11-13
categories: 微服务
tags:
- zuul
- gateway
---
引言：前面一个系列文章介绍了[认证鉴权与API权限控制在微服务架构中的设计与实现](http://blueskykong.com/2017/10/19/security1/) ，好多同学询问有没有完整的demo项目，笔者回答肯定有的。由于之前系列文章侧重讲解了权限前置，所以近期补上完整的后置项目，但是这最好有一个完整的微服务调用。本文主要讲下API网关的设计与实现。netflix-zuul是由netflix开源的API网关，在微服务架构下，网关作为对外的门户，实现动态路由、监控、授权、安全、调度等功能。

## 1. 网关介绍
当使用单体应用程序架构时，客户端（web和移动端）通过向后端应用程序发起一次REST调用来获取数据。负载均衡器将请求路由给N个相同的应用程序实例中的一个。然后应用程序会查询各种数据库表，并将响应返回给客户端。微服务架构下，单体应用被切割成多个微服务，如果将所有的微服务直接对外暴露，势必会出现安全方面的各种问题。   
客户端可以直接向每个微服务发送请求，其问题主要如下：

- 客户端需求和每个微服务暴露的细粒度API不匹配。
- 部分服务使用的协议不是Web友好协议。可能使用Thrift二进制RPC，也可能使用AMQP消息传递协议。
- 微服务难以重构。如果合并两个服务，或者将一个服务拆分成两个或更多服务，这类重构就非常困难了。

如上问题，解决的方法是使用API网关。API网关是一个服务，是系统的唯一入口。从面向对象设计的角度看，它与外观模式类似。API网关封装了系统内部架构，为每个客户端提供一个定制的API。它可能还具有其它职责，如身份验证、监控、负载均衡、限流、降级与应用检测。

![zu](http://ovcjgn2x0.bkt.clouddn.com/zuul-use1-1000x550.jpg "zuul")


## 2. zuul网关
API Gateway，常见的选型有基于 Openresty 的 Kong和基于 JVM 的 Zuul，其他还有基于Go的Tyk。技术选型上，之前稍微调研了Kong，性能还可以。考虑到快速应用和二次开发，netflix-zuul也在Spring Cloud的全家桶中，和其他组件配合使用还挺方便，后期可能还会对网关的功能进行扩增，最后选了Zuul。   


### 2.1 pom配置

```xml
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-zuul</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-consul-discovery</artifactId>
        </dependency>

    </dependencies>
```
在Spring Cloud的项目中，引入zuul的starter，consul-discovery是为了服务的动态路由，这边没有用eureka，是通过注册到consul上的服务实例进行路由。

### 2.2 入口类

```java
@SpringBootApplication
@EnableZuulProxy
public class GatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);

    }
}
```

Spring boot的入口类需要加上`@EnableZuulProxy`，下面看下这个注解。

```java
@EnableCircuitBreaker
@EnableDiscoveryClient
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Import({ZuulProxyConfiguration.class})
public @interface EnableZuulProxy {
}
```
可以看到该注解还包含了`@EnableCircuitBreaker 和 @EnableDiscoveryClient`。`@EnableDiscoveryClient`注解在服务启动的时候，可以触发服务注册的过程，向配置文件中指定的服务注册中心；`@EnableCircuitBreaker`则开启了Hystrix的断路器。

### 2.3 bootstrap.yml

```yml
server:
  port: 10101
    
#spring config
spring:
  application:
    name: gateway-server
  cloud:
    consul:
      discovery:
        preferIpAddress: true
        enabled: true
        register: true
        service-name: api-getway
        ip-address: localhost
        port: ${server.port}
        lifecycle:
          enabled: true
        scheme: http
        prefer-agent-address: false
      host: localhost
      port: 8500

#zuul config and routes
zuul:
  host:
    maxTotalConnections: 500
    maxPerRouteConnections: 50
  routes:
    user:
      path: /user/**
      ignoredPatterns: /consul
      serviceId: user
      sensitiveHeaders: Cookie,Set-Cookie    
```

配置主要包括三块，服务端口，Spring Cloud服务注册，最后是zuul的路由配置。

默认情况下，Zuul在请求路由时，会过滤HTTP请求头信息中的一些敏感信息，默认的敏感头信息通过zuul.sensitiveHeaders定义，包括Cookie、Set-Cookie、Authorization。

zuul.host.maxTotalConnections配置了每个服务的http客户端连接池最大连接，默认值是200。maxPerRouteConnections每个route可用的最大连接数，默认值是20。
### 2.3 支持https
上线的项目一般域名都会改为https协议，顺手写下https的配置。

- 首先申请https的数字证书   
在阿里云生成的针对tomcat服务器CA证书在申请成功后， 下载相应的tomcat证书文件。 包含如下：   
1): *.pfx为keystore文件，服务器用的就是这个文件    
2): pfx-password.txt里包含有keystore所用到的密码    
3): *.key里面包含的是私钥，暂时没用到此文件    
4): *.pem里面包含的是公钥，主要给客户端   

- bootstrap.yml增加如下配置


```yml
# https
server:
  port: 5443
  http: 10101
  ssl:
    enabled: true
    key-store: classpath:214329585620980.pfx
    key-store-password: password
    keyStoreType: PKCS12
```

- 同时支持http和https

```java
@Bean
    public EmbeddedServletContainerFactory servletContainer() {
        TomcatEmbeddedServletContainerFactory tomcat = new TomcatEmbeddedServletContainerFactory() {
            @Override
            protected void postProcessContext(Context context) {
                SecurityConstraint constraint = new SecurityConstraint();
                constraint.setUserConstraint("CONFIDENTIAL");
                SecurityCollection collection = new SecurityCollection();
                collection.addPattern("");
                constraint.addCollection(collection);
                context.addConstraint(constraint);
            }
        };

        tomcat.addAdditionalTomcatConnectors(httpConnector());
        return tomcat;
    }

    @Bean
    public Connector httpConnector() {
        Connector connector = new Connector("org.apache.coyote.http11.Http11NioProtocol");
        connector.setScheme("http");
        //Connector监听的http的端口号
        connector.setPort(httpPort);
        connector.setSecure(false);
        //监听到http的端口号后转向到的https的端口号
        connector.setRedirectPort(securePort);
        return connector;
    }
```

`servletContainer()`把`EmbeddedServletContainerFactory`注入到web容器中，用`postProcessContext`拦截所有的/*请求，并把其关联到下面的httpConnector中。最后，在httpConnector()中，把http设为10101端口，并把http的请求跳转到5443的https端口，这边是读取的配置文件。

至此，至此同时支持https和http的API网关完成，将匹配到/user的请求，路由到user服务，是不是很简单？下面一起深入了解下Zuul。

## 3. 一些internals

internals可以理解为内幕。

### 3.1 过滤器
filter是Zuul的核心，用来实现对外服务的控制。filter的生命周期有4个，分别是pre、route、post、error，整个生命周期可以用下图来表示。

![filter](http://ovcjgn2x0.bkt.clouddn.com/filterzuul.png "zuul过滤器")

一个请求会先按顺序通过所有的前置过滤器，之后在路由过滤器中转发给后端应用，得到响应后又会通过所有的后置过滤器，最后响应给客户端。error可以在所有阶段捕获异常后执行。

一般来说，如果需要在请求到达后端应用前就进行处理的话，会选择前置过滤器，例如鉴权、请求转发、增加请求参数等行为。后面衔接auth系统部分给出具体实现，也是基于pre过滤。

在请求完成后需要处理的操作放在后置过滤器中完成，例如统计返回值和调用时间、记录日志、增加跨域头等行为。路由过滤器一般只需要选择 Zuul 中内置的即可。

错误过滤器一般只需要一个，这样可以在 Gateway 遇到错误逻辑时直接抛出异常中断流程，并直接统一处理返回结果。

### 3.2 配置管理
后端服务 API可能根据情况，有些不需要登录校验了，这个配置信息怎么动态加载到网关配置当中？笔者认为有两种方式：一是配置信息存到库中，定期实现对网关服务的配置刷新；另一种就是基于配置中心服务，当配置提交到配置中心时，触发网关服务的热更新。

后端应用无关的配置，有些是自动化的，例如恶意请求拦截，Gateway 会将所有请求的信息通过消息队列发送给一些实时数据分析的应用，这些应用会对请求分析，发现恶意请求的特征，并通过 Gateway 提供的接口将这些特征上报给 Gateway，Gateway 就可以实时的对这些恶意请求进行拦截。

### 3.3 隔离机制
在微服务的模式下，应用之间的联系变得没那么强烈，理想中任何一个应用超过负载或是挂掉了，都不应该去影响到其他应用。但是在 Gateway 这个层面，有没有可能出现一个应用负载过重，导致将整个 Gateway 都压垮了，已致所有应用的流量入口都被切断？

这当然是有可能的，想象一个每秒会接受很多请求的应用，在正常情况下这些请求可能在 10 毫秒之内就能正常响应，但是如果有一天它出了问题，所有请求都会 Block 到 30 秒超时才会断开（例如频繁 Full GC 无法有效释放内存）。那么在这个时候，Gateway 中也会有大量的线程在等待请求的响应，最终会吃光所有线程，导致其他正常应用的请求也受到影响。

在 Zuul 中，每一个后端应用都称为一个 Route，为了避免一个 Route 抢占了太多资源影响到其他 Route 的情况出现，Zuul 使用 Hystrix 对每一个 Route 都做了隔离和限流。

Hystrix 的隔离策略有两种，基于线程或是基于信号量。Zuul 默认的是基于线程的隔离机制，之前章节的配置可以回顾下，这意味着每一个 Route 的请求都会在一个固定大小且独立的线程池中执行，这样即使其中一个 Route 出现了问题，也只会是某一个线程池发生了阻塞，其他 Route 不会受到影响。

一般使用 Hystrix 时，只有调用量巨大会受到线程开销影响时才会使用信号量进行隔离策略，对于 Zuul 这种网络请求的用途使用线程隔离更加稳妥。

### 3.4 重试机制
一般来说，后端应用的健康状态是不稳定的，应用列表随时会有修改，所以 Gateway 必须有足够好的容错机制，能够减少后端应用变更时造成的影响。

简单介绍下 Ribbon 支持哪些容错配置。重试的场景分为三种：

- okToRetryOnConnectErrors：只重试网络错误
- okToRetryOnAllErrors：重试所有错误
- OkToRetryOnAllOperations：重试所有操作

重试的次数有两种：

- MaxAutoRetries：每个节点的最大重试次数
- MaxAutoRetriesNextServer：更换节点重试的最大次数


一般来说我们希望只在网络连接失败时进行重试、或是对 5XX 的 GET 请求进行重试（不推荐对 POST 请求进行重试，无法保证幂等性会造成数据不一致）。单台的重试次数可以尽量小一些，重试的节点数尽量多一些，整体效果会更好。

如果有更加复杂的重试场景，例如需要对特定的某些 API、特定的返回值进行重试，那么也可以通过实现 RequestSpecificRetryHandler 定制逻辑（不建议直接使用 RetryHandler，因为这个子类可以使用很多已有的功能）。

## 4. 总结
本文首先介绍了API网关的相关知识；其次介绍了zuul网关的配置实现，同时支持https；最后介绍了zuul网关的一些内幕原理，这边大部分参考了网上的文章。网关作为内网与外网之间的门户，所有访问内网的请求都会经过网关，网关处进行反向代理。在整个Spring Cloud微服务框架里，Zuul扮演着”智能网关“的角色。

**github: https://github.com/keets2012/Spring-Boot-Samples/tree/master/api-gateway   
gitee: https://gitee.com/keets/spring-boot-samples/tree/master/api-gateway**


---
### 参考
1. [聊聊 API Gateway 和 Netflix Zuul
](http://www.scienjus.com/api-gateway-and-netflix-zuul/)
2. [Spring Cloud技术分析（4）- spring cloud zuul
](http://tech.lede.com/2017/05/16/rd/server/SpringCloudZuul/)
3.  [netflix-zuul](http://cloud.spring.io/spring-cloud-static/spring-cloud-netflix/1.3.5.RELEASE/single/spring-cloud-netflix.html#_router_and_filter_zuul)

