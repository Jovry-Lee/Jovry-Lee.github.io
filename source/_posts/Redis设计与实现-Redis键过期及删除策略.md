---
title: Redis设计与实现-Redis键过期及删除策略
date: 2020-09-03 14:51:51
tags: ["Redis","Note","Redis设计与实现"]
categories: ["Redis"]
---

### 1 设置键的生存时间或过期时间

通过以下命令，客户端可以以秒或者毫秒精度为数据库中的某个键设置`生存时间`（Time To Live，TTL），在经过指定的秒数或者毫秒数之后，服务器就会自动删除生存时间为0的键。

<!--more-->

- 以`秒`为单位设置过期时间命令：


```
EXPIRE key seconds
```

- 以`毫秒`为单位设置过期时间命令：


```
PEXPIRE key milliseconds
```

- 若过期时间是一个UNIX时间戳，以秒为单位设置过期时间命令:


```
EXPIREAT key timestamp
```

- 若过期时间是一个UNIX时间戳，以微秒为单位设置过期时间命令:


```
PEXPIREAT key milliseconds-timestamp
```

**注：虽然过期的单位及命令不同，但<u>实际上，EXPIRE、PEXPIRE、EXPIREAT三个命令都是使用PEXPIREAT命令来实现的</u>。**



#### 1.1 保存过期时间

redisDb结构的`expires字典`保存了数据库中所有键的过期时间，我们称这个字典为过期字典。

```c
typedef struct redisDb {
    // ...
    // 过期字典，保存着键的过期时间.
    dict *expires;
    // ...
} redisDb;
```

其中，

- 过期字典的`键是一个指针`，这个指针指向键空间中的某个键对象。（也就是某个数据库键）。
- 过期字典的`值是一个long long类型的整数`，这个整数保存了键所指向的数据库键的过期时间——`一个毫秒精度的UNIX时间戳`。



当客户端执行`PEXPIREAT`等命令设置过期时间时，服务器会在数据库的过期字典中关联给定的数据库键和过期时间。



示例：带有过期字典的数据库示意图

![带有过期字典的数据库示意图](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/Redis设计与实现-Redis键过期及删除策略/带有过期字典的数据库示意图.png)



其中，alphabet和book键对象出现了两次，实际上，键空间和过期字典的键都指向同一个键对象，所以并不会出现任何重复对象。



#### 1.2 移除过期时间

通过以下命令可以`移除一个键的过期时间`：

```
PERSIST <key>
```

PERSIST命令就是PEXPIREAT命令的反操作，PERSIST命令在过期字典中查找给定的键，并解除键和值在过期字典中的关联。



#### 1.3 计算并返回剩余生存时间

通过以下命令可以返回键的剩余生存时间：

- ①、以秒为单位返回

```
TTL <key>
```

- ②、以毫秒为单位返回

```
PTTL <key>
```



### 2 过期删除策略

**若一个键过期了，它什么时候会被删除呢？可能情况有哪些？**

- ①、`定时删除`：在设置键的过期时间的同时，创建一个定时器，让定时器在键的过期时间来临时，立即执行对键的删除操作。
- ②、`惰性删除`：放任键过期不管，但每次从键空间中获取键时，都检查取得的键是否过期，吐过过期的话，就删除该键，如果没有过期，则返回该键。
- ③、`定期删除`：每隔一段时间，程序就对数据库进行一次检查，删除里面的过期键。至于要删除多少过期键，以及要检查多少个数据库，则由算法决定。



#### 2.1 定时删除

定时删除策略的**优点**：通过使用定时器，定时删除策略可以保证过期键会尽快地被删除，并释放过期键所占用的内存。(内存友好)

但定时删除策略的**缺点**：它对CPU时间是最不友好的，在过期键比较多的情况下，删除过期键这一行为可能会占用相当一部分CPU时间，在内存不紧张，但CPU时间非常紧张的情况下，将CPU时间用在删除和当前任务无关的过期键上，无疑会对服务器的响应时间和吞吐造成影响。另外，创建一个定时器需要用到Redis服务器中的时间事件，而当前时间事件的实现方式——无序链表，查找一个时间的时间复杂度为O(N)，并不能高效地处理大量时间事件。(CPU不友好)

因此，要让服务器创建大量的定时器，从而实现定时删除策略，现阶段来说并不现实。



#### 2.2 惰性删除

惰性删除的**优点**：对CPU时间友好，程序只会在取出键时才对建进行过期检查，保证删除过期键的操作只会在费做不可的情况下进行，并且删除的目标仅限于当前处理的键，这个策略不会再删除其他无关的国期间上花费任何CPU时间。

惰性删除的**缺点**：对内存不友好，若一个键已经过期，而这个键又仍然留在数据库中，那么只要这个过期键不被删除，他所占用的内存就不会释放。若数据库中有非常多的过期键，而这些过期键又恰好没有访问的话，那么他们也许永远不会被删除（除非用户手动执行`flushdb`），我们甚至可以将这种情况看做`内存泄漏`。



#### 2.3 定期删除

对于定时删除及惰性删除，均有一些明显的缺陷：

- 定时删除占用太多CPU时间，影响服务器的响应时间和吞吐量。
- 惰性删除浪费太多内存，有内存泄露的风险。



定期删除策略是前两种策略的一种整合和折中。

- 定期删除策略每隔一段时间执行一次删除过期键操作，并通过限制删除操作执行的时长和频率来减少删除操作对CPU时间的影响。
- 通过定期删除过期键，定期删除策略有效的减少了因为过期键而带来的内存浪费。



#### 2.4 Redis的过期键删除策略

Redis实际上用的`惰性删除及定期删除策略`，通过配合使用这两种删除策略，服务器可以很好地合理使用CPU时间和避免浪费内存空间之间取得平衡。



##### 2.4.1 惰性删除策略的实现

过期键的惰性删除策略是由`db.c/expireIfNeeded`函数实现的，所有读写数据库的Redis命令在执行前都会调用expireIdNeeded函数对输入键进行检查：

- 若输入键`已经过期`，那么expireIfNeeded函数将输入键从数据库中删除。
- 若输入键`未过期`，那么expireIfNeeded函数不做动作。

注：由于expireIfNeeded函数会将过期键删除，所以每个命令的实现都要能同时处理键存在及键不存在两种情况。



##### 2.4.2定期删除策略的实现

过期键的定期删除策略由`redis.c/activeExpireCycle`函数实现，每当Redis的服务周期性操作`redis.c/serverCron`函数执行时，activeExpireCycle函数就会被调用。

**activeExpireCycle函数的逻辑**：在规定的时间内，分多次遍历服务器中的各个数据库，从数据库的expires字典中`随机检查一部分键的过期时间`，并删除其中的过期键。



### 3 AOF、RDB和复制功能对过期键的处理

#### 3.1 生成RDB文件

执行以下命令将创建一个新的RDB文件，此时，程序会对数据库中的键进行检查，已过期的键不会被保存到新创建的RDB文件中。

- ①、同步保存到磁盘命令

```
SAVE
```

- ②、在后台异步保存当前数据到磁盘命令

```
BGSAVE
```



#### 3.2 载入RDB文件

在启动Redis服务器时，如果服务器开启了RDB功能，那么服务器将对RDB文件进行载入：

- ①、若服务器以**主服务器模式**运行，那么在载入RDB文件时，程序会对文件中保存的键进行检查，未过期的键会被载入到数据库，而过期键则会被忽略，所以过期键对载入RDB文件的主服务器不会造成影响。
- ②、若服务器以**从服务器模式**运行，那么在载入RDB文件时，文件保存的所有键，无论是否过期，都会被载入到数据库中。（*注：因为主从服务器在进行数据同步时，从服务器的数据库就会被清空，所以，一般来说，过期键对载入RDB文件的从服务器也不会造成影响*）



#### 3.3 AOF文件写入

当服务器以**AOF（AppendOnlyFile）持久化模式**运行时，如果数据库中的某个键已经过期，但它还没有被惰性删除或者定期删除，那么AOF文件不会因为这个过期键而产生任何影响。

当过期键被惰性删除或者定期删除之后，程序会向AOF文件追加一条DEL命令，来显式记录该键已经被删除。



示例：若客户端使用`GET message`命令试图访问过期的message键，那么服务器将执行以下三个动作。

- ①、从数据库中删除message键。
- ②、追加一条DEL message命令到AOF文件。
- ③、向执行GET命令的客户端返回空回复。



#### 3.4 AOF重写

在执行AOF重写的过程中，程序会对数据库中的键进行检查，已经过期的键不会被保存到重写后的AOF文件中。因此，数据库中包含过期键不会对AOF重写造成影响。



#### 3.5 复制

当服务器运行在**复制模式**下时，从服务器的过期键删除动作由主服务器控制。

- 当主服务器在删除一个过期键后，会显式地向所有从服务器发送一个DEL命令，告知从服务器删除这个过期键。
- 当从服务器在执行客户端发送的读命令时，即使碰到过期键也不会将过期键删除，而是继续像处理未过期的键一样来处理过期键。
- 从服务器只有在接到主服务器发来的DEL命令之后，才会删除过期键。



**为什么要通过主服务器来控制从服务器统一的删除过期键呢**？

保证主从服务器数据库的一致性。



### 参考资料

1.redis设计与实现（第二版） 黄健宏