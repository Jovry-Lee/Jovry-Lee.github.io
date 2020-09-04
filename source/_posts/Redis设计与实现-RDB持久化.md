---
title: Redis设计与实现-RDB持久化
date: 2020-09-03 15:52:31
tags: ["Redis","Note"]
categories: ["Redis", "Note", "Redis设计与实现"]
---

Redis是内存数据库，它将自己的数据库状态储存在内存里面，若不将其保存到磁盘上，一旦服务器进程退出，服务器中的数据库状态也会消失不见。因此Redis提供了RDB持久化功能，将Redis内存中的数据库状态保存到磁盘里面，避免数据意外丢失。

<!--more-->

RDB持久化可以手动执行，也可以根据服务器配选项定期执行，将某个时间点的数据库状态保存到RDB文件中，也可通过该文件还原生成RDB文件时的数据库状态。

### 1 RDB文件的创建与载入

#### 1.1 RDB文件的创建

Redis通过以下两个命令，可生成RDB文件。

- ①、`同步保存到磁盘命令`

```
SAVE
```

- ②、`在后台异步保存当前数据到磁盘命令`

```
BGSAVE
```

其中，

- `SAVE命令会阻塞Redis服务器进程`，直到RDB文件创建完毕。
- `BGSAVE命令是异步的，它将派生出一个子进程`，然后由子进程负责创建RDB文件，服务器进程（父进程）继续处理命令请求。

创建RDB文件实际工作由`rdb.c/rdbSave函数`完成，SAVE和BGSAVE命令会以不同的方式调用该函数。



BGSAVE以子进程方式执行，在执行时，会拒绝新进入的SAVE、BGSAGE命令。而BGREWRITEAOF命令将会被延迟到BGSAVE命令执行结束。

BGREWRITEAOF命令执行时，新进入的BGSAVE命令将被拒绝。



#### 1.2 RDB文件的载入

RDB`文件的载入工作是在服务器启动时自动执行的`，Redis并没有提供专门用于载入RDB文件的命令。

*注：AOP文件的更新频率通常比RDB文件的更频率高，因此，若服务器开启了AOF持久化功能，那么服务器会优先使用AOF文件来还原数据库状态。只有在AOF持久化功能处于关闭状态时，服务器才会使用RDB文件来还原数据库状态。*



RDB文件载入时，服务器将一直处于阻塞状态，直到载入工作完成为止。



### 2 自动间隔保存

由于BGSAVE命令可以在不阻塞服务器进程的情况下执行，所以Redis允许用户通过设置服务器配置（`/etc/redis/redis.conf`配置文件）的`save`选项，让服务器每隔一段时间自动执行一次BGSAVE命令。



示例：当前Redis默认配置save选项如下，当条件满足时，BGSAVE命令将被执行。

```
# 服务器在900秒之内，对数据库进行了至少1次修改。
save 900 1
# 服务器在300秒之内，对数据库进行了至少10次修改。
save 300 10
# 服务器在60秒之内，对数据库进行了至少10000次修改。
save 60 10000
```



#### 2.1 设置保存条件

当Redis服务启动时，用户可以通过指定配置文件或者传入启动参数的方式设置save选项，若未主动设置save选项，那么服务器将使用redis.conf文件中默认的save条件：

```
save 900 1
save 300 10
save 60 10000
```



在`redisServer结构`中有一个`saveparams`属性，用于保存save设置的保存条件。

```c
struct redisServer {
    // ...
    // 记录了保存条件的数组.
    struct saveparam *saveparams;
    // ...
};
```



默认条件下服务器状态中的`saveparams`数组示意如下：

![默认条件下服务器状态中的saveparams数组示意](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/Redis设计与实现-RDB持久化/默认条件下服务器状态中的saveparams数组示意.png)



#### 2.2 dirty计数器和lastsave属性

- **dirty计数器属性**: 记录距离上一次成功执行`SAVE`或者`BGSAVE`命令之后，服务器对数据状态（服务器中所有数据库）进行了多少次修改（包括写入、删除、更新等操作）。

- **lastsave属性**: 是一个UNIX时间戳，记录了服务器上一次成功执行`SAVE`或者`BGSAVE`命令的时间。



#### 2.3 检查保存条件是否满足

Redis的服务器周期性操作函数serverCron默认每隔`100ms`就执行一次，`该函数用于对正在运行的服务器进行维护，它的其中一项工作就是检查save选项所设置的条件是否满足，若满足，则执行BGSAVE命令。`



示例：服务器状态如下，其中dirty为123，表示距离上一次执行SAVE或者BGSAVE命令后，数据状态修改了123次，此时，若时间来到1378271101时，即301秒后，服务器将自动执行一次BGSAVE命令。执行完成后dirty属性将被清零。

![服务器状态如下，其中dirty为123](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/Redis设计与实现-RDB持久化/服务器状态如下，其中dirty为123.png)



### 3 RDB文件结构

`RDB文件是一个经过压缩的二进制文件，由多个部分组成`，对于不同类型的键值对，RDB文件会使用不同的方式来保存他们。



一个完整的RDB文件包含以下几个部分，如下图所示：

![一个完整的RDB文件包含以下几个部分](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/Redis设计与实现-RDB持久化/一个完整的RDB文件包含以下几个部分.png)

- ①、`REDIS`：长度为5字节的常量，保存着`"REDIS"`5个字符，通过这5个字符，程序在载入文件时，快速检查所在入的文件是否RDB文件。
- ②、`db_version`：长度为4字节，它的值是一个字符串表示整数，记录了RDB文件的版本号，例如“0006”就代表RDB文件的版本为第六版。
- ③、`databases`：包含着零个或任意多个数据库，以及各个数据库中的键值对数据。
- ④、`EOF`：长度为1字节的常量，标志着RDB文件正文内容的结束。
- ⑤、`check_sum`：长度为8字节的无符号整数，保存着通过REDIS、db_version、databases、EOF四个部分计算出的校验和。



#### 3.1 databases部分

一个RDB文件的`databases`部分可以`保存任意多个非空数据库`。

每个非空数据库在RDB文件中都保存着以下三个部分：

![每个非空数据库在RDB文件中都保存着以下三个部分](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/Redis设计与实现-RDB持久化/每个非空数据库在RDB文件中都保存着以下三个部分.png)

- ①、`SELECTDB`：长度为1字节的常量，表示接下来读取的将是一个数据库号码。
- ②、`db_number`：长度为1字节/2字节/5字节的数据库号码。程序根据该号码，调用SELECT命令切换数据库。
- ③、`key_value_pairs`：保存数据库中所有的键值对数据（注：若键值对带有过期时间，那么过期时间也会和键值对保存在一起）。



#### 3.2 key_value_pairs部分

RDB文件的每个`key_value_pairs`：保存数据库中所有的键值对数据（*注：若键值对带有过期时间，那么过期时间也会和键值对保存在一起*）。



##### 3.2.1 不带过期时间的键值对组成

不带过期时间的键值对由以下三个部分组成：

![不带过期时间的键值对由以下三个部分组成](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/Redis设计与实现-RDB持久化/不带过期时间的键值对由以下三个部分组成.png)

- ①、`Type`：记录了value的类型，长度为1字节的常量，包括以下常量，每种常量都代表了一种类型或者底层编码，服务器根据type的值决定如何读入和解释value数据。

- - REDIS_RDB_TYPE_STRING
  - REDIS_RDB_TYPE_LIST
  - REDIS_RDB_TYPE_SET
  - REDIS_RDB_TYPE_ZSET
  - REDIS_RDB_TYPE_HASH
  - REDIS_RDB_TYPE_LIST_ZIPLIST
  - REDIS_RDB_TYPE_SET_INTSET
  - REDIS_RDB_TYPE_ZSET_ZIPLIST
  - REDIS_RDB_TYPE_HASH_ZIPLIST

- ②、`key`：保存了键值对的键对象

- ③、`value`：保存了键值对的值对象。



##### 3.2.2 带过期时间的键值对组成

带过期时间的键值对在RDB文件中的组成如下：

![带过期时间的键值对在RDB文件中的组成](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/Redis设计与实现-RDB持久化/带过期时间的键值对在RDB文件中的组成.png)

其中TYPE、key、value三个部分的意义均与不带过期时间的组成相同。

- ①、EXPIRETIME_MS：长度为1字节的常量，表示接下来读入的是一个以毫秒为单位的过期时间。
- ②、ms：8字节长的带符号的整数，记录了一个以毫秒为单位的UNIX时间戳。



#### 3.3 value编码

RDB文件中的每个value部分都保存了一个值对象，每个值对象的类型都有与之对应的Type记录，根据类型的不同，value部分的结构，长度也会有所不同。



### 4 分析RDB文件

通过Linux的`od指令`来分析Redis服务器产生的RDB文件，使用od命令查看特殊格式的文件内容。通过指定该命令的不同选项可以以十进制、八进制、十六进制和ASCII码来显示文件。RDB文件路径可通过`/etc/redis/redis.conf`文件中查看，如下所示，因此RDB文件路径为：`/var/lib/redis/dump.rdb`



`/etc/redis/redis.conf`中配置如下:

```
# 打印的RDB文件名。
dbfilename dump.rdb
#
# 路径.
dir /var/lib/redis
```



#### 4.1 不包含任何键值对的RDB文件

执行以下命令，创建一个数据库状态为空的RDB文件：

```bash
127.0.0.1:6379> flushall
OK
127.0.0.1:6379> save
OK
```



通过od命令打印RDB文件内容，并使用ASCII或反斜杠序列展示结果(`-c参数`), 如下所示：

```bash
$ od -c dump.rdb
0000000   R   E   D   I   S   0   0   0   6 377 334 263   C 360   Z 334
0000020 362   V
0000022
```

其中：

- 五个字节的“REDIS”字符串
- 四个字节的版本号（0006）
- 377是EOF常量
- 334 263   C 360   Z 334 362   V为8个字节的校验和



#### 4.2 包含字符串键的RDB文件

执行以下命令，分析一个带有单个字符串键的数据库：

```bash
127.0.0.1:6379> flushdb
OK
127.0.0.1:6379> set msg "HELLO"
OK
127.0.0.1:6379> save
OK
```



通过od命令打印RDB文件内容, 并使用ASCII或反斜杠序列展示结果(`-c参数`)：

```bash
$ od -c dump.rdb
0000000   R   E   D   I   S   0   0   0   6 376  \0  \0 003   m   s   g
0000020 005   H   E   L   L   O 377  \n   < 342 005   <   A 217   4
0000037
```

其中:

- 五个字节的“REDIS”字符串
- 四个字节的版本号（0006）
- 376表示SELECTED常量
- \0表示数据库0
- \0表示表示字符串
- 003表示msg的长度
- msg表示key
- 005表示value的长度
- HEELO为value
- 377是EOF常量
-  \n   < 342 005   <   A 217   4为8个字节的校验和



#### 4.3 包含过期时间的字符串键的RDB文件

```bash
127.0.0.1:6379> flushall
OK
127.0.0.1:6379> setex msg 10086 "HELLO"
OK
127.0.0.1:6379> save
OK
```

通过od命令打印RDB文件内容, 并使用ASCII或反斜杠序列展示结果(`-c参数`)：

```bash
$ od -c dump.rdb
0000000   R   E   D   I   S   0   0   0   6 376  \0 374 375   3   {   3
0000020   r 001  \0  \0  \0 003   m   s   g 005   H   E   L   L   O 377
0000040 317   W 312 301 337 313 364 201
0000050
```



#### 4.4 包含一个集合键的RDB文件

```bash
127.0.0.1:6379> flushall
OK
127.0.0.1:6379> sadd lang "C" "JAVA" "RUBY"
(integer) 3
127.0.0.1:6379> SAVE
OK
```

通过od命令打印RDB文件内容, 并使用ASCII或反斜杠序列展示结果(`-c参数`)：

```bash
$ od -c dump.rdb
0000000   R   E   D   I   S   0   0   0   6 376  \0 002 004   l   a   n
0000020   g 003 004   J   A   V   A 004   R   U   B   Y 001   C 377   u
0000040 177 235 372   z 334  \f 216
0000047
```

