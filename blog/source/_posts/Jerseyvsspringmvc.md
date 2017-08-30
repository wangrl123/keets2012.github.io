---
title: Restful Layer of SpringMVC vs Jersey
date: 2017-08-30
categories: Note
tags:
- REST
- Jersey
- Spring
---

笔者项目实现前后端剥离，服务端对外提供restful接口。REST逐渐成为影响Web框架、Web协议与Web应用设计的重要概念。现在有越来越多的公司希望能以简单而又贴合Web架构本身的方式公开Web API，因此REST变得越来越重要也就不足为奇了。使用Ajax进行通信的富浏览器端也在朝这个目标不断迈进。这个架构原则提升了万维网的可伸缩性，无论何种应用都能从该原则中受益无穷。SpringMVC和Jersey都可以为你提供restful风格的接口。本文将介绍SpringMVC中的REST特性并与Jersey进行对比。

 

## 1. REST基础概念

1. 在REST中的一切都被认为是一种资源。
2. 每个资源由URI标识。
3. 使用统一的接口。处理资源使用POST，GET，PUT，DELETE操作类似创建，读取，更新和删除（CRUD）操作。
4. 无状态。每个请求是一个独立的请求。从客户端到服务器的每个请求都必须包含所有必要的信息，以便于理解。
5. 通信都是通过展现。例如XML，JSON。

## 2. Jersey与SpringMVC
JAX-RS（JSR 311）指的是Java API for RESTful Web Services，Roy Fielding也参与了JAX-RS的制订，他在自己的博士论文中定义了REST。对于那些想要构建RESTful Web Services的开发者来说，JAX-RS给出了不同于JAX-WS（JSR-224）的另一种解决方案。目前共有4种JAX-RS实现，所有这些实现都支持Spring，Jersey则是JAX-RS的参考实现。  
   
有必要指出JAX-RS的目标是Web Services开发（这与HTML Web应用不同）而Spring MVC的目标则是Web应用开发。Spring 3为Web应用与Web Services增加了广泛的REST支持，但本文则关注于与Web Services开发相关的特性。我觉得这种方式更有助于在JAX-RS的上下文中讨论Spring MVC。

要说明的第二点是我们将要讨论的REST特性是Spring Framework的一部分，也是现有的Spring MVC编程模型的延续，因此，并没有所谓的“Spring REST framework”这种概念，有的只是Spring和Spring MVC。这意味着如果你有一个Spring应用的话，你既可以使用Spring MVC创建HTML Web层，也可以创建RESTful Web Services层。




