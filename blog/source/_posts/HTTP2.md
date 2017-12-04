---
title: HTTPS vs HTTP 1.1 vs HTTP 2
date: 2017-08-26 11:06:59
categories: Note
tags:
- HTTPS
- HTTP2
---
## 1. HTTPS协议原理分析
### 1.1 需要解决的问题
- 身份验证:确保通信双方身份的真实性。
- 通信加密:通信的机密性、完整性依赖于算法与密钥，通信双方是如何选择算法与密钥的。

### 1.2相关概念
- 数字证书
- CA（certification authority）:数字证书的签发机构。
- HTTPS协议、SSL协议、TLS协议、握手协议的关系
	- HTTPS是Hypertext Transfer Protocol over Secure Socket Layer的缩写，即HTTP over SSL，可理解为基于SSL的HTTP协议。
	- HTTPS协议安全是由SSL协议（目前常用的，本文基于TLS 1.2进行分析）实现的。
	- SSL协议是一种记录协议，扩展性良好，可以很方便的添加子协议，而握手协议便是SSL协议的一个子协议。
	- TLS协议是SSL协议的后续版本，本文中涉及的SSL协议默认是TLS协议1.2版本。   
 
HTTPS协议的安全性由SSL协议实现，当前使用的TLS协议1.2版本包含了四个核心子协议：握手协议、密钥配置切换协议、应用数据协议及报警协议。

### 1.3 握手协议
握手协议的作用便是通信双方进行身份确认、协商安全连接各参数（加密算法、密钥等），确保双方身份真实并且协商的算法与密钥能够保证通信安全。
协议交互图：
![协议交互][xy]  
[xy]:http://img.blog.csdn.net/20170320161103113 "协议交互"

1. ClientHello消息的作用是，将客户端可用于建立加密通道的参数集合，一次性发送给服务端。
2. ServerHello消息的作用是，在ClientHello参数集合中选择适合的参数，并将服务端用于建立加密通道的参数发送给客户端。
3. Certificate消息的作用是，将服务端证书的详细信息发送给客户端，供客户端进行服务端身份校验。
4. ServerKeyExchange消息的作用是，将需要服务端提供的密钥交换的额外参数，传给客户端。有的算法不需要额外参数，则ServerKeyExchange消息可不发送。
5. ServerHelloDone消息的作用是，通知客户端ServerHello阶段的数据均已发送完毕，等待客户端下一步消息。
6. ClientKeyExchange消息的作用是，将客户端需要为密钥交换提供的数据发送给服务端。
7. ChangeCipherSpec消息的作用，便是声明后续消息均采用密钥加密。在此消息后，我们在WireShark上便看不到明文信息了。
8. Finished消息的作用，是对握手阶段所有消息计算摘要，并发送给对方校验，避免通信过程中被中间人所篡改。

### 1.4 总结
HTTPS如何保证通信安全，通过握手协议的介绍，我们已经有所了解。
但是，在全面使用HTTPS前，我们还需要考虑一个众所周知的问题——HTTPS性能。
相对HTTP协议来说，HTTPS协议建立数据通道的更加耗时，若直接部署到App中，势必降低数据传递的效率，间接影响用户体验。

## 2. HTTP 2
### 2.1 HTTP1.x协议

随着互联网的快速发展，HTTP1.x协议得到了迅猛发展，但当App一个页面包含了数十个请求时，HTTP1.x协议的局限性便暴露了出来：

- 每个请求与响应需要单独建立链路进行请求(Connection字段能够解决部分问题)，浪费资源。
- 每个请求与响应都需要添加完整的头信息，应用数据传输效率较低。
- 默认没有进行加密，数据在传输过程中容易被监听与篡改。

### 2.2 HTTP 2介绍
HTTP2正是为了解决HTTP1.x暴露出来的问题而诞生的。

> 说到HTTP2不得不提spdy。    
由于HTTP1.x暴露出来的问题，Google设计了全新的名为spdy的新协议。spdy在五层协议栈的TCP层与HTTP层引入了一个新的逻辑层以提高效率。spdy是一个中间层，对TCP层与HTTP层有很好的兼容，不需要修改HTTP层即可改善应用数据传输速度。    
spdy通过多路复用技术，使客户端与服务器只需要保持一条链接即可并发多次数据交互，提高了通信效率。 
而HTTP2便士基于spdy的思路开发的。    
通过流与帧概念的引入，继承了spdy的多路复用，并增加了一些实用特性。

新特性：

- 多路复用
- 压缩头信息
- 对请求划分优先级
- 支持服务端Push消息到客户端

HTTP2目前在实际使用中，只用于HTTPS协议场景下，通过握手阶段ClientHello与ServerHello的extension字段协商而来，所以目前HTTP2的使用场景，都是默认安全加密的。

查看了wiki发现：
> Netty and OkHttp are the only two implementations supported by Spring. 

### 2.3 协议协商
HTTP2协议的协商是在握手阶段进行的。

协商的方式是通过握手协议extension扩展字段进行扩展，新增Application Layer Protocol Negotiation字段进行协商。

在握手协议的ClientHello阶段，客户端将所支持的协议列表填入Application Layer Protocol Negotiation字段，供服务端进行挑选。

### 2.4 多路复用Multipexing
在HTTP2中，同一域名下的请求，可通过同一条TCP链路进行传输，使多个请求不必单独建立链路，节省建立链路的开销。

为了达到这个目的，HTTP2提出了流与帧的概念，流代表请求与响应，而请求与响应具体的数据则包装为帧，对链路中传输的数据通过流ID与帧类型进行区分处理。下图是多路复用的抽象图，每个块代表一帧，而相同颜色的块则代表是同一个流。
![http2_stream][dl]  
[dl]:http://img.blog.csdn.net/20170320161954297 "http2 stream"

归纳下okhttp的多路复用实现思路：

1. 通过请求的Address与连接池中现有连接Address依次匹配，选出可用的Connection。
2. 通过Http2xStream创建的FramedStream在发送了请求后，将FramedStream对象与StreamID的映射关系缓存到FramedConnection中。
3. 收到消息后，FramedConnection解析帧信息，在Map中通过解析的StreamID选出缓存的FramedStream，并唤醒FramedStream进行Response的处理。

### 2.5 压缩头信息
HTTP2为了解决HTTP1.x中头信息过大导致效率低下的问题，提出的解决方案便是压缩头部信息。具体的压缩方式，则引入了HPACK。

HPACK压缩算法是专门为HTTP2头部压缩服务的。为了达到压缩头部信息的目的，HPACK将头部字段缓存为索引，通过索引ID代表头部字段。客户端与服务端维护索引表，通信过程中尽可能采用索引进行通信，收到索引后查询索引表，才能解析出真正的头部信息。

HPACK索引表划分为动态索引表与静态索引表，动态索引表是HTTP2协议通信过程中两端动态维护的索引表，而静态索引表是硬编码进协议中的索引表。

作为分析HPACK压缩头信息的基础，需要先介绍HPACK对索引以及头部字符串的表示方式。

索引

索引以整型数字表示，由于HPACK需要考虑压缩与编解码问题，所以整型数字结构定义下图所示：
![索引结构][sy]
[sy]:http://img.blog.csdn.net/20170320161738200 "索引结构"

- 类别标识:
通过类别标识进行HPACK类别分类，指导后续编解码操作，常见的有1，01，01000000等八个类别。
- 首字节低位整型:
首字节排除类别标识的剩余位，用于表示低位整型。若数值大于剩余位所能表示的容量，则需要后续字节表示高位整型。
- 结束标识:
表示此字节是否为整型解析终止字节。
- 高位整型:
字节余下7bit，用于填充整型高位。

