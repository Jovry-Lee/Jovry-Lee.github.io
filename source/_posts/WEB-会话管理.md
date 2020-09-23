---
title: WEB-会话管理
date: 2020-09-23 16:18:34
tags: ["WEB","SESSION","COOKIE","TOKEN","JWT"]
categories: ["WEB"]
---

### １引入

首先，我们需要知道，HTTP是无状态协议，即HTTP协议对于发送过的请求/接收过的响应均不作持久化处理。`HTTP（ HyperText Transfer Protocol）`，顾名思义，超文本传输协议。最开始基本上就是文档的浏览而已，作为服务器，不需要记录谁在某一段时间里都浏览了什么文件；还有就是为了能更快的处理大量事务，确保协议的可伸缩性，特意将其设计的如此简单。
<!--more-->
但是随着交互式Web应用兴起，如在线购物网站，需要登录的网站等，就马上面临一个问题，即管理会话，必须记住哪些人登陆了系统，哪些人往自己的购物车加购了，且必须将每个人区分开。因此就引入了Session、Cookie、Token等技术。

### 2 管理会话方法

我们需要知道，会话管理的目的是为了验证用户身份。

#### 2.1 使用session做会话管理

**session是基于cookie实现的会话管理机制**，session存储在服务器端，sessionId会被存储到客户端的cookie中。
<u>（注：禁用cookie，seesion也将失效。）</u>

##### 2.1.1 管理方式

给所有的用户发一个会话标识（Session id），其实就是一个随机字符串，每个用户的字符串不一样，当用户发起请求时，将该字符串带过来，就能区分谁是谁了。

##### 2.1.2 问题

使用这种管理方式，虽然每个人只需要保存自己的Session id，但服务器需要记住所有用户Session id，这对服务器是很大的开销。并且严重限制了服务器的扩展能力，比如，使用两个机器组成一个集群，某个用户小F，通过机器A登录了系统，其Session is会保存在机器A上，假设下一次请求被转发到机器B，而机器B上没有小F的Session。
　　

##### 2.1.3 解决方法

- `Session Stick（Session 粘黏）`：让小F的请求一直粘黏在机器A上。但此时若机器A挂了，还是需要转机器B上，无法从根本上解决问题。
- `Session复制`：把Session id在两台机器上复制，但这样比较麻烦。
- `将Session统一存储起来`：把Session id集中存储到一个地方，所有的机器都来访问这个地方的数据，这样就不需要复制了，但这种方式可能增加了单点失败的风险，若负责Session的机器挂了，则所有用户都需要重新登陆。

##### 2.1.4 Session的生命周期

Session保存在服务端，为了获取更高的存取速度，服务器一般会把Session放在内存中，每个用户都会有一个独立的Session，如果Session里面的内容太过复杂，当大量的用户访问服务器时，可能会导致内存溢出，所以Session的内容应该适当的精简。
　　
当第一次访问服务器时，服务器会创建并维护这个Session。当用户访问一次服务器，无论是否有操作Session，服务器都会认定这个Session活跃了一次，当越来越多的用户访问服务器时，Session就会越来越多。为了防止内存溢出，服务器会把长时间没有活跃的Session删除。这个时间就是Session的过期时间，超过了该时间，Session就会自动失效。
　　

#### 2.2 使用Token（令牌）做会话管理

<img src="https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/HTTP/使用token做令牌管理.png" alt="使用token做令牌管理" style="zoom:80%;" />

##### 2.2.1 管理方式

对数据做一个签名，比如使用HMAC-SHA256算法，加上一个服务器才知道的密钥，对数据做一个签名，把这个签名和数据一起作为token，由于别人不知道密钥，就无法伪造token。
　　
生成的Token服务器不需要保存，当下一次小F请求来的时候，再使用相同的算法进行签名计算，若相同，则判定小F已经登陆过了，并且可以直接取到小F的User id，如果不相同，则数据部分肯定被修改了，就告诉发送者，对不起，没有认证。

（**思想：使用CPU的计算时间换取Session的存储空间**）

##### 2.2.2 问题

若某个用户的token被别人盗取，服务器也会认为这是一个合法的用户（注：Session也存在这个问题），<u>其安全性一般是通过https协议来保证传输的安全性。</u>

##### 2.2.3 Token的有效期

Token的认证机制可以完成自身的验证，根据token的实际应用场景，对token进行存储，比如登录场景，需要进行一系列精细化控制，如权限、token有效性、自身注销、非法用户的拦截操作等。

设置有效期需要从需求和用户体验触发，尽可能寻找一个平衡，保证用户体验的前提下，尽可能短。但若用户在操作时token过期需要重新登陆，体验将会很糟。
　　
**为了解决在操作过程中不让用户感受到token失效的问题，通常有以下两种方案**：

- **方案一**：在服务端保存token状态，用户每次操作都会刷新（推迟）token的过期时间（Session就是采用这种策略来保持用户登录状态的）。
  - 问题：在前后端分离、单页App这些情况下，每秒钟可能发起多个请求，每次都去刷新过期时间会产生非常大的代价。若token持久化到数据库或文件中，代价就更大了。所以通常为了提升效率，减少消耗，会把token的过期时间保存在缓存或内存中。

- **方案二**：使用Refresh Token，可以避免频繁的读写操作。这种方案，服务器不需要刷新token的过期时间，一旦token失效，就反馈给前端，前端使用Refesh token申请一个全新的token继续使用。该种方案中，服务端只需要在客户端请求更新token的时候，对Refresh token的有效性检查，大大减少了更新有效期的操作，也避免了频繁读写。当然Refresh Token也是有有效期的，但这个有效期就可以长一些。如下图所示：

​    ![refresh_token](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/HTTP/refresh_token.png)

#### 2.3 JWT

JWT（JSON Web Token）是目前最流行的跨域认证解决方案，<u>是一种认证授权机制</u>。

**JWT的原理是**：服务器认证以后，生成一个JSON对象，发回给用户，后续用户需要与服务端通信时，需要发回这个JSON对象。服务器完全只靠这个对象认定用户身份，且为了防止数据被篡改，生成对象时会添加一个签名。

实际JWT的数据结构示例如下：

```
eyJleHAiOjE1OTkwMzIyNzksImtpZCI6M30.eyJkZXZpY2VfaWQiOiI1ZGJhZDkxZTE3OTgwOWVmYTZjYjc1ZmI3MmE2NzRlYyIsInNtX2RldmljZV9pZCI6IiIsImRpZ2l0YWxfZGV2aWNlX2lkIjoiIn0.fob_tbEagrBm3o3kK4qFdPw5tdJQG9uIQeg9ryCnIpk                   
```

​       

### 3 Session、Cookie、Token的区别

#### 3.1 Cookie

Cookie是一个非常具体的东西，<u>指的就是浏览器里面能永久的数据</u>。仅仅是浏览器实现的一种数据存储功能。

Cookie由服务器生成，发送给浏览器，浏览器把Cookie以key-value形式保存在某个目录下的文本文件内，下一次请求同一网站时把该Cookie发送给服务器。通过对Cookie的解析，服务器可以识别请发送者的身份。由于Cookie是存在客户端，所以浏览器加入了一些限制确保Cookie不会被恶意使用，同时不会占据太多磁盘空间，所以<u>每个域的Cookie数量是有限的</u>。

*注：Cookie是不可跨域名的，隐私安全机制禁止网站非法获取其他网站的Cookie。*

（一般来说，同一个一级域名下的两个二级域名也不能交互使用Cookie。比如，a.test.com和b.test.com，因为二者的域名不同。若想要test.com下的二级域名都可以使用该Cookie，需要设置CookieDomain参数为.test.com，这样a.test.com和b.test.com就能访问同一个Cookie。）



**缺陷**：

- Cookie的数量和长度都是有限的
- 潜在的安全风险：Cookie可能被截取篡改，如果Cookie被拦截，就可能会获取到所有的信息。
- `用户可能会配置禁用浏览器或客户端设备接收Cookie的能力，因此可能会限制这一功能`。
- 有些状态不可能保保存在客户端。



#### 3.2 Session

Session，顾名思义，会话。服务器要知道当前请求是谁发送来的，为了做这种区分，服务器就要给每个客户端分配不同的“身份标识”，在客户端每次向服务器发送请求时，都带上这个“身份标识”，服务器就知道这个请求来自谁了。（<u>客户端通常会以Cookie的方式保存SessionId</u>）。

服务器使用Session把用户信息临时存在服务器上，用户离开网站后SessionId会被销毁。这种用户信息存储方式相对于Cookie来说更安全。

**缺陷**：若Web服务器做了负载均衡，那么下一个操作请求到了另一台服务器的时候Session会丢失。



#### 3.3 Token

在大多数使用Web API的互联网公司中，token是多用户下处理认证的最佳方式。使用Cpu计算时间换取Session的存储空间。

##### 3.3.1 特点

- 无状态、可扩展
- 支持移动设备
- 跨程序调用
- 安全



##### 3.3.2 基于Token的身份验证过程

基于Token的身份验证过程如下：

![token身份验证](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/HTTP/token身份验证.png)

- 用户通过用户名和密码发送请求
- 程序验证
- 程序返回一个签名的token给客户端
- 客户端存储token，并且用于每次发送请求
- 服务器待验证token并返回数据

------

### 参考资料

[彻底理解session、cookie、token](https://www.cnblogs.com/moyand/p/9047978.html)
[阿里面试题cookie和session的区别及session的生命周期](https://zhuanlan.zhihu.com/p/80716402)
[大话Token、Cookie和Session](https://zhuanlan.zhihu.com/p/88185448)
[还分不清 Cookie、Session、Token、JWT？](https://zhuanlan.zhihu.com/p/164696755)
[JSON Web Token 入门教程](https://www.ruanyifeng.com/blog/2018/07/json_web_token-tutorial.html)
[RFC1579（JWT）](https://tools.ietf.org/html/rfc7519)