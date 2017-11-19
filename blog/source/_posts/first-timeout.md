---
title: Spring Cloud 服务第一次请求超时的优化
date: 2017-11-18
categories: 微服务
tags:
- Spring Cloud
- Ribbon
---
## 1. 问题背景
使用Spring Cloud组件构建的服务集群，在第一次请求时经常会出现timeout的情况，然而第二次就正常了。Spring Cloud版本为Dalston.SR4。

启动涉及到的相关服务：

- gateway(zuul网关)
- auth-Service（鉴权服务）
- user-Service（用户服务）

测试的端点接口为：http:/login/oauth/token。服务之间的调用顺序为：gateway->auth-Service->user-Service。网关收到客户端的请求，转发请求到鉴权服务，鉴权服务对用户身份的核验是通过调用用户服，用户服务给鉴权服务返回身份校验的结果，鉴权服务将身份授权信息返回给gateway，gateway将最终的结果response返回给客户端。   
三个服务启动后，通过zipkin监控调用链路信息，可以看到第一次和第二次调用情况如下图所示：

![not-eager-1st](http://ovcjgn2x0.bkt.clouddn.com/gw-not-eager.jpg "首次调用端点")

![second](http://ovcjgn2x0.bkt.clouddn.com/gw-second.jpg "第二次调用信息")

通过上面两次的链路监控信息截图，可以看到第一次的耗时是第二次的10多倍。遇到某些情况，很可能会出现第一次请求的超时。去官网看了下，主要原因是zuul网关和各个调用服务之间的Ribbon进行客户端负载均衡的Client懒加载，导致第一次的请求调用包括了创建Ribbon Client的时间。通过启动日志信息就可以发现：

![lazy-load](http://ovcjgn2x0.bkt.clouddn.com/lazy-load.jpg "Ribbon 客户端懒加载")

下面分两部分解决这个问题，一是服务之间调用Ribbon的饥饿加载，对应上面的测试为auth-Service调用user-Service；二是zuul网关的饥饿加载。

## 2. ribbon的饥饿加载

经过调查发现，造成第一次auth-Service调用user-Service耗时长的原因主要是，Ribbon进行客户端负载均衡的服务实例并不是在服务启动的时候就初始化好的，而是在调用的时候才会去创建相应的服务实例。所以第一次调用user-Service耗时不仅仅包含发送HTTP请求的时间，还包含了创建Ribbon Client的时间，这样一来如果创建时间速度较慢，同时设置的请求超时又比较短的话，很容易就会出现耗时很长甚至超时的情况。
在官网可以看到如下的配置说明：

> Each Ribbon named client has a corresponding child Application Context that Spring Cloud maintains, this application context is lazily loaded up on the first request to the named client. This lazy loading behavior can be changed to instead eagerly load up these child Application contexts at startup by specifying the names of the Ribbon clients.

意为Spring Cloud为每个Ribbon客户端维护了一个相对的子应用环境的上下文，应用的上下文在第一次请求到指定客户端的时候懒加载。不过可以通过如下配置进行修改：

```yml
ribbon:
  eager-load:
    enabled: true
    clients: client1, client2, client3
```
按照如上的配置之后，发现鉴权服务启动时就将user服务的Ribbon客户端进行了加载。

![el](http://ovcjgn2x0.bkt.clouddn.com/eager-load.jpg "user服务eager load")


## 3. zuul网关的饥饿加载
上面小节解决了auth-Service调用user-Service的Ribbon客户端启动时饥饿加载。网关作为对外请求的入口，zuul内部使用Ribbon调用其他服务，Spring Cloud默认在第一次调用时懒加载Ribbon客户端。zuul同样需要维护一个相对的子应用环境的上下文，所以也需要启动时饥饿加载。

> Zuul internally uses Ribbon for calling the remote url’s and Ribbon clients are by default lazily loaded up by Spring Cloud on first call. This behavior can be changed for Zuul using the following configuration and will result in the child Ribbon related Application contexts being eagerly loaded up at application startup time.

具体配置如下：

```yml
zuul:
  ribbon:
    eager-load:
      enabled: true
```

至此，优化完成，再次重启服务进行第一次请求，发现情况已经好多了，大家可以自己动手尝试改进一下。

## 4. 总结
本文主要介绍了Spring Cloud的服务第一次请求超时的优化方法。首先介绍了问题的背景，并排查了问题造成的原因，主要是Ribbon客户端的懒加载；然后分别针对zuul网关和服务之间调用的Ribbon客户端进行配置，使其启动时便加载Ribbon客户端的相关上下文信息。最后想说的是，http调用毕竟还是性能远低于RPC。。🙂

---
### 参考
1. [spring-cloud Dalston.SR4](http://cloud.spring.io/spring-cloud-static/Dalston.SR4/single/spring-cloud.html)
2. [Spring Cloud实战小贴士：Ribbon的饥饿加载(eager-load)模式](http://blog.didispace.com/spring-cloud-tips-ribbon-eager/)
