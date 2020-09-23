---
title: Architecture-既然有HTTP请求，为何要用RPC调用
date: 2020-09-23 16:56:17
tags:
  - Architecture
categories:
  - Architecture
---

### 1 引言

之前在一次面试中，面试官提到服务之间的调用，为什么要选择RPC？HTTP也能实现服务之间的通信，为什么不使用HTTP呢？或者说RPC比Http好在哪里？

<!--more-->

### 2 RPC

#### 2.1 什么是RPC？

RPC(Remote Procedure Call，远程过程调用)，是指计算机程序在不同的地址空间执行时，其编码方式与本地过程调用类似，无需显式编码远程交互的详细信息[*参考资料3*]。
<u>RPC只是一种概念、一种设计，用于解决不同服务之间的调用问题</u>，通常包含传输协议（*可以直接建立在TCP之上，也可以建立在HTTP协议之上*）和序列化协议这两个。

#### 2.2 RPC解决的问题

RPC主要解决的问题是：让分布式或微服务系统中不同服务之间的调用向本地调用一样简单[*参考资料2*]。

#### 2.3 RPC的基本流程

**RPC的基本流程图**如下：

<img src="https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/Architecture/RPC基本流程.png" alt="RPC基本流程" style="zoom:80%;" />

一次完整的RPC调用流程如下：

- 1）client调用以本地调用方式调用服务； 
- 2）client stub接收到调用后负责将方法、参数等组装成能够进行网络传输的消息体； 
- 3）client stub找到服务地址，并将消息发送到服务端； 
- 4）server stub收到消息后进行解码；
- 5）server stub根据解码结果调用本地的服务；
- 6）本地服务执行并将结果返回给server stub； 
- 7）server stub将返回结果打包成消息并发送至消费方； 
- 8）client stub接收到消息，并进行解码；
- 9）服务消费方得到最终结果。 

<u>RPC框架的目标就是要2~8这些步骤都封装起来，让用户对这些细节透明。</u>
　　
**时序图如**下：

<img src="https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/Architecture/RPC调用时序图.png" alt="RPC调用时序图" style="zoom:80%;" />

#### 2.4 服务注册&发现

​    ![服务注册-发现](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/Architecture/服务注册-发现.png)

- Server：暴露服务的服务提供方。
- Client：调用远程服务的服务消费方。
- Register：服务注册与发现的注册中心。



服务提供者启动后想注册中心注册机器IP、Port以及提供服务列表；

服务消费者启动时向注册中心获取服务提供方提供地址列表，可实现软负载均衡以及故障转移、



### 3 HTTP

#### 3.1 什么是HTTP？

HTTP（Hypertext Transfer Protocol，超文本传输协议）是一种应用协议，用于分布式、协作、超媒体信息系统。



### 4 RPC和HTTP的关系是什么

**RPC是一种概念，HTTP是RPC实现的一种方式，至于为什么要这么用，因为业务场景不同。**

- 对于HTTP：为应用层提供服务，需要支持web网页或其他请求的各种诉求，本身数据报文包含的内容较多，有很多的头信息。

- 对于RPC：可直接使用TCP、UDP等协议，在提供服务上更简洁，报文更小，性能更快，可专注于提高远程服务调用。



### 5 对于微服务使用RPC调用的好处

- 调用简单，真正提供了类似于调用本地方方法一样调用接口的功能。
- 参数返回值简单明了，自定义返回结果，无需二次解析。
- 轻量，没有多余的信息
- 便于管理，基于注册中心。
- 解耦，通过RPC解耦服务，

------

参考资料：

[微服务调用为啥用RPC框架，Http不更简单吗？](https://developer.51cto.com/art/201904/595840.htm)
[服务之间的调用 HTTP代替RPC？](http://blog.itpub.net/31559985/viewspace-2676573/)
[Remote procedure call](https://en.wikipedia.org/wiki/Remote_procedure_call)
[RPC框架面试总结-RPC原理及实现](http://www.360doc.com/content/18/1116/23/99071_795390393.shtml)
[Hypertext Transfer Protocol](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol)