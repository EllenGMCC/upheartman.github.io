---
title: HTTP协议
categories: 
- 读书笔记
- 图解HTTP
---

# 请求报文

```
GET /index.htm HTTP/1.1 
Host: hackr.jp
```

起始行开头的GET表示请求访问服务器的类型，称为方法 （method）。随后的字符串` /index.htm` 指明了请求访问的资源对象， 也叫做请求 URI（request-URI）。最后的 HTTP/1.1，即 HTTP 的版本 号，用来提示客户端使用的 HTTP 协议功能。

综合来看，这段请求内容的意思是：请求访问某台 HTTP 服务器上的`/index.htm `页面资源。

**请求报文是由请求方法、请求 URI、协议版本、可选的请求首部字段 和内容实体构成的。**

![](https://xiaoflyfish.oss-cn-beijing.aliyuncs.com/image/20210525002946.png)

# 响应报文

基本上由协议版本、状态码（表示请求成功或失败的数字代 码）、用以解释状态码的原因短语、可选的响应首部字段以及实体主 体构成

![](https://xiaoflyfish.oss-cn-beijing.aliyuncs.com/image/20210525003322.png)

# 无状态协议

HTTP 是一种不保存状态，即无状态（stateless）协议。HTTP 协议自 身不对请求和响应之间的通信状态进行保存。也就是说在 HTTP 这个 级别，协议对于发送过的请求或响应都不做持久化处理

使用 HTTP 协议，每当有新的请求发送时，就会有对应的新响应产 生。协议本身并不保留之前一切的请求或响应报文的信息。这是为了 更快地处理**大量事务**，确保协议的可伸缩性，而特意把 HTTP 协议设 计成如此简单的。

HTTP/1.1 虽然是无状态协议，但为了实现期望的保持状态功能，于 是引入了 Cookie 技术。有了 Cookie 再用 HTTP 协议通信，就可以管 理状态了。

# 请求URI定位资源

HTTP 协议使用 URI 定位互联网上的资源。正是因为 URI 的特定功 能，在互联网上任意位置的资源都能访问到

