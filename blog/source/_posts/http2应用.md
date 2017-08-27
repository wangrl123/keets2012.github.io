---
title: HTTP 2实际应用
date: 2017-08-27
categories: Note
tags:
- Java
- HTTP2
---
## 1. 背景介绍
### 1.1 需要解决的问题
本文来源于项目需要，项目所使用微服务框架为Spring Cloud，微服务之间的调用基于HTTP 1.X协议，上一篇文章 [HTTPS vs HTTP 1.1 vs HTTP 2](http://hacloud.club/2017/08/26/HTTP2/)，介绍了http2 和http1.1的相关知识，也列出了http1.1局限性，链路不能复用、数据不加密、头信息过多等等。为此，笔者在想能不能将feign client的调用基于http2协议，做了如下调研。
  
HTTP/2 源自 SPDY/2。SPDY 系列协议由谷歌开发，于 2009 年公开。它的设计目标是降低 50% 的页面加载时间。当下很多著名的互联网公司，例如百度、淘宝、UPYUN 都在自己的网站或 APP 中采用了 SPDY 系列协议（当前最新版本是 SPDY/3.1），因为它对性能的提升是显而易见的。主流的浏览器（谷歌、火狐、Opera）也都早已经支持 SPDY，它已经成为了工业标准，HTTP Working-Group 最终决定以 SPDY/2 为基础，开发 HTTP/2。   
2013年8月，进行首次测试，诞生的时间很晚，笔者搜索了网上关于http2实践的相关信息，发现并不多。

### 1.2 关于项目介绍
Spring Cloud是笔者项目采用的微服务框架，具体介绍见[Spring Cloud](http://hacloud.club/2017/07/18/spring-cloud/)。Spring Cloud是基于Spring Boot开发的组合框架，Spring Boot内置的容器是Tomcat，笔者的项目一般都会exclude Tomcat的引用，使用的是Jetty容器。所以搜索的主题词就变成了 jetty http2。

## 2. 调研结果
大部分的人习惯于将Tomcat运行在8080端口，再用Apache server在前面提供https。这样做是因为简单且验证过的方法。使用http2 ，你将被迫使用https，这样就不用部署Apache (or nginx)。

### 2.1 服务端
> Currently Jetty and undertow are the only servers in Spring Boot that support HTTP/2.   
Jetty has booked some progress and this repository shows an excellent example. In my opinion it’s still too much custom code, but they’re getting there.   
The next candidate is undertow. It seems almost too easy, but it works. Because we use AJP in our current configuration it even means this HTTP/2 solution has less lines of code!

当前Spring Boot只有Jetty 和 undertow支持HTTP/2。 [样例repo](https://github.com/bclozel/http2-experiments/)是一个很好的example。
总得分为三步：

1. update dependencies
	- org.springframework.boot:spring-boot-starter-undertow
	- org.mortbay.jetty.alpn:alpn-boot:8.1.8.v20160420
2. create a servlet container bean
```
	@Bean   
	UndertowEmbeddedServletContainerFactory embeddedServletContainerFactory() {      
    		UndertowEmbeddedServletContainerFactory factory = new    	UndertowEmbeddedServletContainerFactory();   
    	factory.addBuilderCustomizers(   
            	builder -> builder.setServerOption(UndertowOptions.ENABLE_HTTP2,    	true));   
    	return factory;   
	}
```
3. start your server with alpn
为了启动服务，需要带上 -Xbootclasspath 参数来包括alpn 。因为alpn 有可能在jdk中没有。
```
-Xbootclasspath/p:/home/harrie/.m2/repository/org/mortbay/jetty/alpn/alpn-boot/8.1.8.v20160420/alpn-boot-8.1.8.v20160420.jar
```

### 2.2 客户端
> Currently Java HTTP/2 clients are scarce. According to this wiki Netty and OkHttp are the only two implementations supported by Spring. To switch HTTP-client in RestTemplate you have to call the constructor with a different ClientHttpRequestFactory (either Netty4ClientHttpRequestFactory or OkHttpClientHttpRequestFactory). 

当前Java的http2的客户端也很少，Spring只有Netty and OkHttp支持。这边我们选用了OkHttp，因为OkHttp本来就有在feign client中内置。

```
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-web</artifactId>
    <version>4.3.0.RC1</version>
</dependency>
 
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-core</artifactId>
    <version>4.3.0.RC1</version>
</dependency>
 
<dependency>
    <groupId>com.squareup.okhttp3</groupId>
    <artifactId>okhttp</artifactId>
    <version>3.2.0</version>
</dependency>
```
### 2.3 浏览器对于HTTP/2的支持
通过[浏览器支持http2](http://caniuse.com/#search=http2)查看。

![HTTP/2的支持][ch]
[ch]:http://ovci9bs39.bkt.clouddn.com/browser%20for%20http2.png "http2 support"

### 2.4 okhttp
目前, Http/1.1在全世界大范围的使用中, 直接废弃跳到http/2肯定不现实. 不是每个用户的浏览器都支持http/2的, 也不是每个服务器都打算支持http/2的, 如果我们直接发送http/2格式的协议, 服务器又不支持, 那不是挂掉了! 总不能维护一个全世界的网站列表, 表示哪些支持http/2, 哪些不支持?   
为了解决这个问题, 从稍高层次上来说, 就是为了更方便地部署新协议, HTTP/1.1 引入了 Upgrade 机制. 这个机制在 RFC7230 的「6.7 Upgrade」这一节中有详细描述.   
简单说来, 就是先问下你支持http/2么? 如果你支持, 那么接下来我就用http/2和你聊天. 如果你不支持, 那么我还是用原来的http/1.1和你聊天.

1. 客户端在请求头部中指定Connection和Upgrade两个字段发起 HTTP/1.1 协议升级. HTTP/2 的协议名称是 h2c, 代表 HTTP/2 ClearText.
2. 如果服务端不同意升级或者不支持 Upgrade 所列出的协议，直接忽略即可（当成 HTTP/1.1 请求，以 HTTP/1.1 响应）.
3. 如果服务端同意升级，那么需要这样响应

> HTTP/1.1 101 Switching Protocols   
Connection: Upgrade   
Upgrade: h2c   
[ HTTP/2 connection ... ]

HTTP Upgrade 响应的状态码是 101，并且响应正文可以使用新协议定义的数据格式。

这样就可以完成从http/1.1升级到http/2了. 同样也可以从http/1.1升级到WebSocket.
OkHttp使用了请求协议的协商升级, 无论是1.1还是2, 都先只以1.1来发送, 并在发送的信息头里包含协议升级字段. 接下来就看服务器是否支持协议升级了. OkHttp使用的协议升级字段是ALPN, 如果有兴趣, 可以更深入的查阅相关资料.

## 3. 总结

总体看来，现在Spring boot 是可以支持HTTP/2 server和client。现有项目的api接口面向移动端和web端，web浏览器对于http2的支持在上文已经说明。

----
参考资料：   
[OkHttp使用完全教程](http://www.jianshu.com/p/ca8a982a116b)   
[Spring Boot with HTTP/2 – Start a server and make REST calls as a client](https://vanwilgenburg.wordpress.com/2016/04/01/spring-boot-http2/)   
[HTTPS 与 HTTP2 协议分析](http://blog.csdn.net/zhangzq86/article/details/64907340)

