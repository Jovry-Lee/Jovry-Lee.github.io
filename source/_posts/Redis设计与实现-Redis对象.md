---
title: Redis设计与实现-Redis对象
date: 2020-08-31 17:14:48
tags: ["Redis","Note"]
categories: ["Redis", "Note", "Redis设计与实现"]
---



Redis基于其数据结构（例如，SDS、双端链表、字典、压缩列表、整数集合等）创建了一个对象系统，该系统包含`字符串对象`、`列表对象`、`哈希对象`、`集合对象`和`有序集合`对象这五种对象，每种对象都用到了至少一种数据结构。

<!--more-->

# 1 对象的类型与编码

Redis使用`对象`来表示数据库中的健和值，每次当我们在Redis的数据库中新创建一个键值对时，我们至少会创建两个对象，一个对象用作键值对的健（键对象），另一个对象用作键值对的值（值对象）。



Redis中每个对都由`redisObject`结构表示，如下所示：

```c
typedef struct redisObject {
    // 类型.
    unsigned type:4;
    // 编码.
    unsigned encoding:4;
    // 记录了对象最后一次被命令程序访问的时间。
    unsigned lru:REDIS_LRU_BITS; /* lru time (relative to server.lruclock) */
    // 引用次数
    int refcount;
    // 指向底层实现数据结构的指针。
    void *ptr;
} robj;
```



## 1.1 类型

对象的type属性记录了对象的类型，这个类型包括以下5中类型：

- **REDIS_STRING**：字符串对象(type命令输出：“string”)
- **REDIS_LIST**：列表对象(type命令输出：“list”)
- **REDIS_HASH**：哈希对象(type命令输出：“hash”)
- **REDIS_SET**：集合对象(type命令输出：“set”)
- **REDIS_ZSET**：有序集合对象(type命令输出：“zset”)

*使用`type命令`可以返回**数据库键对应的值对象类型***



*对于Redis数据库保存的`键值`对来说，<u>键总是一个字符串对象</u>，而值则可以是字符串对象、列表对象、哈希对象、集合对象或者有序集合对象中的一种，因此我们称“XX键”表示这个数据库键所对应的值为XX对象。*



## 1.2 编码和底层实现

对象的`ptr`指向对象的底层实现数据结构，而这些数据结构由对象的`encoding`属性决定。

encoding对象属性记录了对象所使用的编码，也就是说这个对象使用了什么数据结构作为对象的底层实现，其对象的编码如下：

- **REDIS_ENCODING_INT**：long类型的整数(object encoding命令输出：int)
- **REDIS_ENCODING_EMBSTR**：embstr编码的简单动态字符串(object encoding命令输出：embstr)
- **REDIS_ENCODING_RAW**：简单动态字符串(object encoding命令输出：raw)
- **REDIS_ENCODING_HT**：字典(object encoding命令输出：hashtable)
- **REDIS_ENCODING_LINKEDLIST**：双端链表(object encoding命令输出：linkedlist)
- **REDIS_ENCODING_ZIPLIST**：压缩列表(object encoding命令输出：ziplist)
- **REDIS_ENCODING_INTSET**：整数集合(object encoding命令输出：intset)
- **REDIS_ENCODING_SKIPLIST**：跳跃表和字典(object encoding命令输出：skiplist)



每种类型的对象都至少使用了两种不同的编码，如下所示：

![每种类型的对象都至少使用了两种不同的编码](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/Redis设计与实现-Redis对象/每种类型的对象都至少使用了两种不同的编码.png)

*使用`object encoding命令`可以**查看一个数据库键的值对象的编码***



**为什么Redis要使用encoding属性来设定对象所使用的编码，而不是为特定类型的对象关联一种固定的编码？**

因为使用encoding属性设定编码方式可以<u>根据不同的适用场景设置不同的编码，从而优化对象在某一场景下的效率。</u>例如，在列表对象包含的元素比较少时，Redis使用压缩列表作为列表对象的底层实现，压缩列表比双向链表更节约内存，且在元素数量较少是，在内存中以连续块方式保存的压缩列表比起双向链表可以更快被载入到缓存中。随着列表对象包含的元素越来越多，是用压缩列表来保存元素的优势逐渐消失，对象就会将底层实现从压缩列表转向功能更强、也更合适保存大量元素的双端链表上。

# 2 字符串对象

字符串对象的编码可以是`int`、`raw`、`embstr`。

- **int**：字符串对象保存的是`整数值，并且这个整数值可以用long类型类表示`，那么字符串对象会将整数值保存在字符串对象结构的ptr属性里面（将void *装换成long），并将字符串对象的编码设置为int。
- **raw**：字符串对象保存的是一个`字符串值，并且这个字符串值的长度大于32字节`，那么字符串对象将使用一个简单动态字符串（SDS）来保存这个字符串值，并将对象的编码设置为raw。
- **embstr**：字符串对象保存的是一个`字符串值，并且这个字符串值的长度小于等于32字节`，那么字符串对象将使用embstr编码的方式来保存这个字符串值。

示例：

```bash
127.0.0.1:6379> set number 10086
OK
127.0.0.1:6379> object encoding number
"int"

127.0.0.1:6379> set story "Long, long ago there lived a king and a queen, they have a beautiful daughter..."
OK
127.0.0.1:6379> strlen story
(integer) 80
127.0.0.1:6379> object encoding story
"raw"

127.0.0.1:6379> set msg "hello"
OK
127.0.0.1:6379> object encoding msg
"embstr"
```



**问: 既然有raw编码方式，为什么还要有embstr编码呢？它们有什么异同点？**

- 相同点:

  - embstr编码是专门用于保存短字符串的一种优化编码方式，这种编码和raw一样，都是用redisObject结构和sdshdr结构来表示字符串对象。

- 不同点：

  - raw编码会<u>调用两次内存分配函数</u>分别创建redisObject结构和sdshdr结构。

  - embstr则通过调用一次内存分配函数来分配<u>一块连续的空间</u>，空间中依次包含redisObject和sdshdr两个结构。

    

    embstr编码创建的内存块结构如下：

![embstr编码创建的内存块结构](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/Redis设计与实现-Redis对象/embstr编码创建的内存块结构.png)



**问: 使用embstr编码的字符串对象来保存短字符串值有什么好处？**

- ①、embstr编码将创建的字符串对象所需的内存分配次数从raw编码的两次降低为一次。
- ②、释放embstr编码的字符串对象只需要调用一次内存释放函数，而释放raw编码的字符串对象需要调用两次内存释放函数。
- ③、embstr编码的字符串对象的所有数据都保存在一块连续的内存里面，所以这种编码的字符串对象比raw编码的字符串对象能更好的利用缓存带来的优势。



注：`long double类型表示的浮点数在Redis中也是作为字符串值来保存的`, 如下示例所示：

```bash
127.0.0.1:6379> set pi 3.14
OK
127.0.0.1:6379> object encoding pi
"embstr"
```



下图为字符串对象保存各类型值的编码方式：

![字符串对象保存各类型值的编码方式](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/Redis设计与实现-Redis对象/字符串对象保存各类型值的编码方式.png)



## 2.1 编码转换

`int编码`的字符串对象和`embstr编码`的字符串对象在条件满足的情况下，会被转换为`raw编码`的字符串对象。

- **int编码->raw编码**：执行一些命令，使得对象保存的值不再是整数值，而是一个字符串值时。例如APPEND命令。
- **embstr编码->raw编码**：Redis并没有为embstr编码提供任何修改程序，因此`实际上embstr编码的字符串是只读的`，当对embstr编码的字符串进行修改时，程序会先将对象的编码从embstr转换为raw，再执行修改命令。



示例1：

```bash
127.0.0.1:6379> set number 10086
OK
127.0.0.1:6379> object encoding number
"int"
127.0.0.1:6379> append number " is a good number!" # 使用append命令将对象变为一个字符串.
(integer) 23
127.0.0.1:6379> get number
"10086 is a good number!"
127.0.0.1:6379> object encoding number
"raw"
```

示例2：

```bash
127.0.0.1:6379> set msg "hello word"
OK
127.0.0.1:6379> object encoding msg
"embstr"
127.0.0.1:6379> append msg " again!" # embstr字符串是只读的,当对其进行修改时,将强制转换为raw编码,再进行修改.
(integer) 17
127.0.0.1:6379> object encoding msg
"raw"
```



# 3 列表对象

列表对象的编码可以是`ziplist`或者`linkedlist`。

- ziplist编码的列表对象: 使用`压缩列表`作为底层实现，每个压缩列表结点(entry)保存了一个列表元素。
- linkedlist编码的列表对象: 使用`双端链表`作为底层实现，每个双端链表结点都保存了一个字符串对象，而每个字符串对象都保存了一个列表元素。



示例1：

```bash
127.0.0.1:6379> rpush numbers 1 "three" 5
(integer) 3
```

`ziplist编码`的numbers列表对象如下：

![ziplist编码的numbers列表对象](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/Redis设计与实现-Redis对象/ziplist编码的numbers列表对象.png)



linkedlist编码的numbers列表对象如下：

![linkedlist编码的numbers列表对象](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/Redis设计与实现-Redis对象/linkedlist编码的numbers列表对象.png)



其中完整的StringObject表示方式如下：

![完整的StringObject表示方式](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/Redis设计与实现-Redis对象/完整的StringObject表示方式.png)

## 3.1 编码转换

当列表对象可以同时满足以下两个条件时，列表对象使用`ziplist`编码。其他情况下需要使用`linkedlist`编码。

- ①、列表对象保存的所有字符串元素的长度都小于64字节（由配置`list-max-ziplist-value`决定）；
- ②、列表对象保存的元素数量小于512个（由配置`hash-max-ziplist-entries`决定）。



示例1：保存长度太大的元素而进行编码转换的情况。

```bash
127.0.0.1:6379> rpush blah "hello" "world" "again"
(integer) 3
127.0.0.1:6379> object encoding blah
"ziplist"
127.0.0.1:6379> rpush blah "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
(integer) 5
127.0.0.1:6379> object encoding blah
"linkedlist"
```



示例2：保存的元素数量过多而进行编码转换的情况。

```bash
# 俩比偶对象包含512个元素。
127.0.0.1:6379> eval "for i=1, 512 do redis.call('rpush', KEYS[1], i)end" 1 "integers"
(nil)
# 获取列表长度.
127.0.0.1:6379> llen integers
(integer) 512
127.0.0.1:6379> object encoding integers
"ziplist"

# 再向列表对象推入一个新元素，使得对象保存的元素数量达到513个。
127.0.0.1:6379> rpush integers 512
(integer) 513
127.0.0.1:6379> object encoding integers
"linkedlist"
```



# 4 哈希对象

哈希对象的编码可以是`ziplist`或者`hashtable`。

- **ziplist编码**：ziplist编码作为哈希对象使用`压缩列表`作为底层实现，每当有新的键值对要加入到哈希对象时，程序会先将保存了键的压缩列表节点推入到压缩列表表尾，然后再将保存了值的压缩列表节点推入到压缩列表表尾。因此，`保存了同一键值对的两个结点总是紧挨在一起的，键结点在前，值结点在后`。

- **hashtable编码**: 哈希对象使用`字典`作为底层实现，哈希对象中的每个键值对都是用一个字典键值对来保存：字典中每个键/值都是一个字符串对象，对象中保存了键值对的键/值。



示例：

```bash
127.0.0.1:6379> hset profile name "Tom"
(integer) 1
127.0.0.1:6379> hset profile age 25
(integer) 1
127.0.0.1:6379> hset profile career "Programmer"
(integer) 1
```



ziplist编码的profile哈希对象的底层实现如下：

![ziplist编码的profile哈希对象的底层实现](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/Redis设计与实现-Redis对象/ziplist编码的profile哈希对象的底层实现.png)



hashtable编码的profile哈希对象底层实现如下：

![hashtable编码的profile哈希对象底层实现](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/Redis设计与实现-Redis对象/hashtable编码的profile哈希对象底层实现.png)

## 4.1 编码转换

**问: 什么情况下，哈希对象使用ziplist编码？**

当哈希对象可以同时满足以下两个条件时，哈希对象使用ziplist编码：

- ①、哈希对象保存的所有键值对的键和值的字符串长度都小于64字节（`hash-max-ziplist-value`）；
- ②、哈希对象保存的键值对数量小于512个（`hash-max-ziplist-entries`）；

当以上两个条件任何一个不满足时，都会进行编码装换为hashtable。



示例1：键值对的键太大而引起的编码转换。

```bash
127.0.0.1:6379> hset book name "Mastering C++ in 21 days"
(integer) 1
127.0.0.1:6379> object encoding book
"ziplist"
127.0.0.1:6379> hset book long_long_long_long_long_long_long_long_long_long_long_long_long_long_description "content"
(integer) 1
127.0.0.1:6379> object encoding book
"hashtable"
```



示例2： 哈希对象因为包含的键值对数量过多而引起编码转换。

```bash
127.0.0.1:6379> EVAL "for i=1, 512 do redis.call('HSET', KEYS[1], i, i)end" 1 "numbers"
(nil)
127.0.0.1:6379> hlen numbers
(integer) 512
127.0.0.1:6379> object encoding numbers
"ziplist"
127.0.0.1:6379> hmset numbers "key" "value"
OK
127.0.0.1:6379> object encoding numbers
"hashtable"
```



# 5 集合对象

集合对象的编码可以是`intset`或者`hashtable`。

- intset编码的集合对象: 使用`整数集合`作为底层实现，集合对象包含的所有元素都被存在整数集合里。
- hashtable编码的集合对象: 使用`字典`作为底层实现，`字典的每个键都是一个字符串对象，每个字符串对象包含了一个集合元素，而字典的值则全部被设置为null。`



示例：

```bash
# intset
127.0.0.1:6379> sadd numbers 1 3 5
(integer) 3
127.0.0.1:6379> object encoding numbers
"intset"

#hashtable
127.0.0.1:6379> sadd fruits "apple" "banana" "cherry"
(integer) 3
127.0.0.1:6379> object encoding fruits
"hashtable"
```



intset编码的numbers集合对象

![intset编码的numbers集合对象](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/Redis设计与实现-Redis对象/intset编码的numbers集合对象.png)

hashtable编码的fruits集合对象

![hashtable编码的fruits集合对象](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/Redis设计与实现-Redis对象/hashtable编码的fruits集合对象.png)

## 5.1 编码的转换

**问: 什么情况下，集合对象使用intset编码？**

当集合对象同时满足以下两个条件时，对象使用intset编码：

- ①、结合兑现个保存的所有元素都是整数值；
- ②、集合对象保存的元素数量不超过512个（`set-max-intset-entries`）。

当其中一个条件不满足时，intset编码方式将自动变为hashtable编码方式。



示例：

```bash
127.0.0.1:6379> sadd numbers 1 3 5
(integer) 3
127.0.0.1:6379> object encoding numbers
"intset"
127.0.0.1:6379> sadd numbers "seven" # 添加非整数元素, 使得集合编码变为为hashtable.
(integer) 1
127.0.0.1:6379> object encoding numbers
"hashtable"
```



# 6 有序集合对象

有序集合的编码可以是`ziplist`或者`skiplist`。

- **ziplist编码**：ziplist编码的压缩列表对象使用`压缩列表`作为底层实现，每个集合元素使用两个紧挨在一起的压缩列表节点来保存，`第一个节点保存元素的成员，第二个元素保存元素的分值`。`压缩列表内的集合元素按分值从小到大进行排序`，分值较小的元素被放置在靠近表头的方向，分值较大的元素则被放置在靠近表尾的方向。

- **skiplist编码**：skiplist编码的有序集合对象使用`zset结构`作为底层实现，`一个zset结构同时包含一个字典和一个跳跃表`。



zset结构如下：

```c
typedef struct zset {
    // 该字典为有序集合创建了一个从成员到分值的映射，字典中的每个键值对都保存了一个集合元素，字典的键保存了元素的成员，字典的值则保存了元素的分值。
    dict *dict;
    // 按分值从小到大保存了所有集合的元素。
    zskiplist *zsl;
} zset;
```

*注：有序集合每个元素的成员都是一个字符串对象，而每个元素的分值都是一个double类型的浮点数。<u>跳跃表和字典均是通过指针来共享元素的成员和分值，因此同时使用跳跃表和字典来保存集合元素不会产生任何重复的成员或者分值，也不会因此而浪费额外的内存</u>。*



示例：

```bash
127.0.0.1:6379> zadd price 8.5 apple 5.0 banana 6.0 cherry
(integer) 3
```



ziplist编码的price对象：

![ziplist编码的price对象](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/Redis设计与实现-Redis对象/ziplist编码的price对象.png)



skiplist编码的price对象：

![skiplist编码的price对象](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/Redis设计与实现-Redis对象/skiplist编码的price对象.png)



**问: 为什么有序集合需要同时使用跳跃表和字典来实现？**

理论上，有序集合可以单独使用字典或者跳跃表的其中一个数据结构来实现，但无论单独使用字典还是跳跃表，其性能上对比同时视同字典和跳跃表都会有所降低。

- 若单独使用字典来实现有序集合，那么虽然查找时时间复杂度保留，仍然是O(1)，但是当执行范围操作时，程序都要需要对所有元素进行排序操作。而完成排序操作至少需要O(NlogN)时间复杂度，以及额外的O(N)内存空间用于保存排序后的数组。
- 若单独使用跳跃表来实现有序集合，那么跳跃表执行范围型操作的所有优点都会被保留，但因为没有了字典，所以根据成员查找分值这一操作的复杂度将从O(1上升为O(logN)。



# 7 内存回收

Redis在自己的对象系统中构建了一个`引用计数`技术实现内存回收机制，通过这一机制，程序可以通过跟踪对象的引用计数信息，在适当的时候自动释放对象，并进行内存回收。

每个对象的计数信息由`redisObject结构`中有一个`refcount属性`记录：

```c
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:REDIS_LRU_BITS; /* lru time (relative to server.lruclock) */
    // 引用计数。
    int refcount;
    void *ptr;
} robj;
```



对象的引用计数信息会随着对象的使用状态而不断变化：

- ①、在创建一个新对象时，引用计数的值会被初始化为1；
- ②、当对象被一个新程序使用时，它的引用计数值会被赠一；
- ③、当对象不再被一个程序引用时，它的引用计数值会被减一；
- ④、当对象的引用计数值变为0时，对象所占用的内存会被释放掉。

通过以下命令可以查键对应的值对象的引用计数：

`object refcount xxx`



示例：

```
127.0.0.1:6379> set a 100
OK
127.0.0.1:6379> object refcount a
(integer) 2
```

**问: 为什么此处引用计数是为2呢？**

因为对象引用计数属性还带有对象共享的作用，redis在初始化服务器时，创建一万个字符对象，这些对象包括从0到9999的所有整数值，当服务器需要用到这些值时，服务器就会使用这些共享对象，而不是新创建对象。



此处持有这个值对象的两个程序分别是，其示意图如下：

- ①、这个值对象的服务器程序；
- ②、共享这个值对象的键a；

![持有同一值对象的两个程序](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/Redis设计与实现-Redis对象/持有同一值对象的两个程序.png)

注：Redis会共享值为0~9999的字符串对象。



**问: 为什么Redis不共享包含字符串的对象？**

当服务器服务器考虑将一个共享对象设置为键的值对象时，程序需要先根据给定的共享对象和键想创建的目标对象是否完全相同，只有在共享对象和目标对象完全相同的情况下，程序才会将共享对象用作键的值对象，而一个共享对象保存的值越复杂，验证共享对象和目标对象是否相同所需的复杂度就越高，消耗的CPU时间也会越多。

- 若共享对象是保存的整数值的字符串对象，那么验证操作的复杂度就是O(1).
- 若共享对象是保存字符串值的字符串对象，那么验证操作的复杂度为O(N)。
- 若共享对象是包含多个值（或者对象的）对象，比如列表对象或者哈希对象，那么验证操作的复杂度将会是O(N^2)。

因此，尽管共享更复杂的对象可以节约更多的内存，但是收到CPU时间的限制，Redis值对包含整数值的字符串对象进行共享。



# 8 对象的空转时长

redisObject结构还包含一个`lru属性`，该属性`用于记录对象最后一次被命令程序访问的时间`：

```c
typedef struct redisObject {
    // 类型.
    unsigned type:4;
    // 编码.
    unsigned encoding:4;
    // 记录了对象最后一次被命令程序访问的时间。
    unsigned lru:REDIS_LRU_BITS; /* lru time (relative to server.lruclock) */
    // 引用次数
    int refcount;
    // 指向底层实现数据结构的指针。
    void *ptr;
} robj;
```

`空转时长`：通过当前时间减去键的值对象的lru时间计算得出的，通过以下命令可打印出给定键的空转时长：

`object idletime xxx`

*注：该命令的实现是特殊的，执行时并不会修改值的lru属性。*



示例：

```bash
127.0.0.1:6379> set msg "hello world"
OK
127.0.0.1:6379> object idletime msg
(integer) 8
# 访问msg键的值
127.0.0.1:6379> get msg
"hello world"
# 键处于活跃状态，空转时长为0.
127.0.0.1:6379> object idletime msg
(integer) 0
```

空转时长还有一项作用，若服务器打开了`maxmemory`选项，并且服务器用于回收内存算法为volatile-lru或者allkeys-lru，那么当服务器占用的内存数超过了maxmemory选项所设置的上限值时，空转时长较高的那么部分键会优先被服务器释放，从而回收内存。



# 参考资料

1.redis设计与实现（第二版） 黄健宏