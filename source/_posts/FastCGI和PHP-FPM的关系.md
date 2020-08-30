---
title: FastCGI和PHP-FPM的关系
date: 2020-08-25 10:11:05
tags: ["PHP"]
categories: ["PHP"]
---

#### 1 相关概念

##### 1.1 CGI

`CGI`（Common Gateway Interface, 通用网关接口）是`WEB服务器`与`WEB Application`进行通信的工具，用于保证WEB Server传递的数据是标准格式的，是一种协议，其程序运行在服务器上，CGI可以用任何一种语言编写，只要该语言具有标准输入、输出和环境变量，如PHP、Perl等。

<!--more-->

<u>WEB Server只是内容的分发者</u>。CGI规定要传输哪些数据，以什么格式传递给后方处理。

- 若客户端请求的是**静态数据**（如：`/index.html`），那么WEB Server会取文件系统中找到这个文件，并发送给浏览器，其流程如下所示：
![客户端请求静态数据](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/FastCGI和PHP-FPM的关系/客户端请求静态数据.png)

- 若请求的是**非静态数据**（如：`/index.php`），根据配置文件，WEB Server（如：Nginx）知道这个不是静态文件，那么Nginx将会把该请求简单处理后交给PHP解析器。比如：URL、查询字符串、POST数据、HTTP Header等。PHP解析器收到数据首先会解析php.ini文件，初始化执行环境，然后处理请求，再以CGI规定的格式返回处理后的结果，退出进程。WEB Server再把结果返回给浏览器。其流程如下所示：

![客户端请求动态数据](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/FastCGI和PHP-FPM的关系/客户端请求非静态数据.png)

**缺陷**：<u>CGI只是个协议，与进程没有关系，PHP解析器在每次请求都会解析php.ini文件，初始化执行环境。所以处理时间耗时较长。</u>



##### 1.2 FastCGI

`FastCGI`（Fast Common Gateway Interface，快速通用网关接口）是一种让交互程序与WEB服务器通信的协议。FastCGI是CGI的增强版，通过常驻（Long-live）进程解决了CGI的Fork-Execute的缺点，减少Web服务器与CGI程序之间交互的开销，从而使服务器可以同时处理更多的网页请求。

**FastCGI的实现**：FastCGI通过FastCGI进程管理器管理进程，在其自身初始化时，会fork多个FastCGI进程，并等待WEB服务器的连接。当一个请求来时，Web服务器将环境变量和页面请求通过socket或TCP连接传递给FastCGI进程。响应通过相同的连接从进程返回到WEB服务器，再传递给客户端。

（*注：每次请求连接可能在相应结束时关闭，但WEB服务器和FastCGI服务进程都将持续，不会销毁。*）

![FastCGI处理过程](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/FastCGI和PHP-FPM的关系/FastCGI处理过程.png)

**FastCGI优点**：

- ①、每个单独的FastCGI进程在其生命周期内可以处理多个请求，从而避免每个请求进程创建和终止的开销。

- ②、并发处理多个请求稳定性和可扩展性（配置多个FastCGI服务器）。



**FastCGI的缺点**：

- 由于FastCGI是多进程的，所以相比CGI多进程消耗更多服务器内存（如：PHP-CGI解释器每进程消耗7~25M内存，将这个数乘以50或100将是很大的内存数。）



##### 1.3 PHP-CGI

`php-cgi`是早期php官方出品的FastCGI管理器。

缺点：

①、PHP-CGI更改了php.ini配置后需要重启php-cgi才能让配置生效，不支持平滑重启。

②、直接杀死PHP-CGI进程，PHP就不能运行了。而PHP-FPM和SPAWN-FCGI在杀掉进程后，守护进程会平滑重新生成子进程。

③、不支持动态worker调度，只能一开始指定要起几个worker。



##### 1.4 PHP-FPM

PHP-CGI是使用PHP语言实现的了FastCGI的一个程序，后来被PHP官方收了。

PHP-FPM加入了动态调度功能，可以根据请求来访压力变化动态增加worker进程数量来支持reload指令，让worker进程在完成当前请求后重启，并应用php.ini新配置



#### 2 Web Server传递数据给Web Application（PHP应用）的方法

PHP使用SAPI提供的2种连接方法与WEB Server通信：`mod_php`和`mod_fastcgi`。其中Apache通过mod_php来解析PHP，Nginx通过mod_fastcgi来解析。

##### 2.1 Mod_php模式

mod_php通过嵌入PHP解释器到Web Server进程中，只能与WEB Server配合使用。以Apache为例，其调用PHP执行过程如下：

![Mod_php处理过程](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/FastCGI和PHP-FPM的关系/Mod_php模式.png)

`Apache->httpd->php5_module->sapi->php`

**缺点**：

- ①、该种模式下，WEB服务器每接收一个请求，都会产生一个进程通过sapi接连PHP完成请求，若用户过多，并发量大时，服务器会承受不住。

- ②、内存占用大，无论是否用到PHP解释器都会将其加在到内存中，典型的就是处理CSS、JS之类的静态文件是完全没有必要加载解释器。

- ③、把mod_php编进Web服务器时，出问题很难定位是php的问题还是WEB服务器的问题。



**2.2 Mod_FastCGI模式**

`mod_fastcgi`以独立的进程形式出现，只要对应的WEB Server（如：Nginx）实现了cgi/fastCgi协议，就能处理PHP请求。



##### 2.3 Apache（php_mod）与Nginx()性能对比【参考资料7】

- ①、若仅在Web服务器上运行PHP，那么Apache相较Nginx有更高的性能，若看到明显的性能差异，则应检查AllowOverride是否打开（httpd.conf），然后重试。

- ②、若运行混合内容（例如添加CSS,JS和图像），则Nginx将提供更好的总体性能，但不会更快地运行PHP。另外Nginx在拒绝服务攻击及减轻CDN风险做的更好。

#### 3 总结

简单的来说，`PHP-FPM`是实现了`FastCGI协议`的一个`进程管理器`。它通过常驻内存的方式解决了早期CGI实现每次都需要解析php.ini及初始化执行环境导致的耗时长的问题，支持平滑启动及动态Worker调度。



#### 参考资料

1. [搞不清FastCGI与PHP-FPM之间是个什么样的关系？](https://segmentfault.com/q/1010000000256516)

2. [概念了解：CGI，FastCGI，PHP-CGI与PHP-FPM](http://www.nowamagic.net/librarys/veda/detail/1319)
3. [在Nginx中理解和实现FastCGI代理](https://www.digitalocean.com/community/tutorials/understanding-and-implementing-fastcgi-proxying-in-nginx)

4. [php中fastcgi和php-fpm是什么东西](https://zhuanlan.zhihu.com/p/101433025)

5. [FastCGI](https://zh.wikipedia.org/wiki/FastCGI)
6. [php-cgi和php-fpm有什么关系？](https://segmentfault.com/q/1010000008356979)

7. [WHY IS FASTCGI /W NGINX SO MUCH FASTER THAN APACHE /W MOD_PHP?](https://www.eschrade.com/page/why-is-fastcgi-w-nginx-so-much-faster-than-apache-w-mod_php/)
8. [CGI、FastCGI和PHP-FPM关系图解](https://www.awaimai.com/371.html)