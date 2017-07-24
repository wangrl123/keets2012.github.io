---
title: REST与HTTP
date: 2017-07-10 16:06:59
categories: Note
tags:
- REST
- HTTP
---
## 幂等性
	Methods can also have the property of "idempotence" in that (aside from error or expiration issues) the side-effects of N > 0 identical requests is the same as for a single request.

- 安全操作与幂指相等特性（Safety /Idempotence）
HTTP 的 GET、HEAD 请求本质上应该是安全的调用，即：GET、HEAD 调用不会有任何的副作用，不会造成服务器端状态的改变。对于服务器来说，客户端对某一 URI 做 n 次的 GET、HAED 调用，其状态与没有做调用是一样的，不会发生任何的改变。
- HTTP 的 PUT、DELTE 调用，具有幂指相等特性 , 即：客户端对某一 URI 做 n 次的 PUT、DELTE 调用，其效果与做一次的调用是一样的。HTTP 的 GET、HEAD 方法也具有幂指相等特性。
HTTP 这些标准方法在原则上保证你的分布式系统具有这些特性，以帮助构建更加健壮的分布式系统。

- 当然作为设计的基础，几个必须的原则还是要遵守的：
  1. 当标准合理的时候遵守标准。
  2. API应该对程序员友好，并且在浏览器地址栏容易输入。
  3. API应该简单，直观，容易使用的同时优雅。
  4. API应该具有足够的灵活性来支持上层ui。
  5. API设计权衡上述几个原则。
 
 ## HTTP
http请求由三部分组成，分别是：请求行、消息报头、请求正文.

- http 1.1/2

  http://www.blogjava.net/yongboy/archive/2015/03/23/423751.html

- HTTP/1.1，HTTP客户端无法重试非幂等请求，尤其在错误发生的时候，由于无法检测错误性质这会对重试带来不利的影响。
  1. HTTP/2不允许使用连接特定头部字段
  2. 新增的5个头部
  3. 推送机制的一些特性需求
  4. RST_STREAM等帧标志位的使用


