---
title: MySQL-并发控制
date: 2020-09-10 16:54:48
tags: ["MySQL","锁","高性能MySQL","死锁","一致性非锁定读"]
categories: ["MySQL", "锁"]
---

无论何时，只要有多个查询需要在同一时刻修改数据，都会产生并发控制的问题．通常并发控制是采用`锁的方式`来防止数据不一致．

锁是数据库系统区别于文件系统的一个关键特性．`锁机制用于管理对共享资源（临界资源）的并发访问`．

<!--more-->

#### 1、读写锁

在处理并发读和并发写时，可以通过实现一个由两种类型的锁组成的锁系统来解决，分别是：  
- ①、共享锁（shared lock）——读锁， 多个客户在同一时刻读取同一个资源，而互不干扰。
- ②、排它锁（exclusive lock）——写锁， 写锁会阻塞其他的写锁和读锁。



#### 2、锁粒度

一种提高共享资源并发性的方式就是让锁定对象更具有选择性．尽量只锁定需要修改的部分数据，而不是所有的资源．更理想的方式是只对对修改的数据片进行精确锁定．`任何时候，在给定的资源上，锁定的数据量越少，并发度越高．`

问题是，加锁需要消耗资源．因此需要寻求一种策略，`在锁的开销和数据安全性之间做平衡，这就是锁策略`．



每种数据库及相同数据库下不同引擎的锁策略都不同，例如：

- MySQL的`MyISAM存储引擎，其锁是表锁设计`．在并发情况下的读没有问题，但是并发写时性能就会差一些．
- 对于Microsoft SQL Server数据库，2005版本之前是采用的页锁，相对于MyISAM引擎来说，并发性有所提高，但是对于热点数据页的并发问题，仍然无能为力．2005版本之后，通过支持乐观并发和悲观并发，在乐观并发下开始支持行级锁（其实现方式与InnoDB存储引擎完全不同）．
- InnoDB引擎锁的实现与Oracle数据库类似，提供`一致性非锁定读，行级锁支持`．



Mysql提供了多钟锁策略，每种Mysql`存储引擎都可以实现自己的锁策略和锁粒度`。
锁定的资源越少，系统并发程度越高，但加锁消耗的资源越大（获得锁，检查锁，释放锁等操作）。



##### 2.1、表锁（table lock）

表锁是Mysql中最基本的锁策略，并且是开销最小的锁策略。它将会锁定整张表。其写锁比读锁有更高的优先级，因此一个写锁请求可能会被插入到读锁请求队列的前面．

某些情况下，Mysql自身会使用表锁实现目的，并忽略存储引擎的锁机制．例如，ALTER TABLE之类的语句使用的表锁。



##### 2.2、行级锁（row lock）

行级锁可以最大程度地支持并发处理，但同时也带来了最大锁开销。`行级锁在存储引擎层实现的`。



#### 3、InnoDB的锁类型

每种Mysql存储引擎都可以实现自己的锁策略和锁粒度，InnoDB作为MySQL的默认引擎，拥有７种类型的锁用于并发控制．



InnoDB共有七种类型的锁：

- ①、`共享锁/排它锁（Shared and Exclusive Locks）`
- ②、`意向锁（Intention Locks）`
- ③、`记录锁（Record Locks）`
- ④、`间隙锁（Gap Locks）`
- ⑤、`临键锁（Next-key Locks）`
- ⑥、`插入意向锁（Insert Intention Locks）`
- ⑦、`自增锁（Auto-inc Locks）`



##### 3.1 共享锁/排它锁（Shared and Exclusive Locks）

在InnoDB里实现了`标准的行级锁(row-level locking)`，共享/排它锁：  
 (1)事务拿到某一行记录的共享S锁，才可以读取这一行；  
 (2)事务拿到某一行记录的排它X锁，才可以修改或者删除这一行；

其兼容互斥表如下：

|      | S    | X    |
| ---- | ---- | ---- |
| S    | 兼容 | 互斥 |
| X    | 互斥 | 互斥 |

即：
- ①、多个事务可以拿到一把S锁，读读可以并行；
- ②、而只有一个事务可以拿到X锁，写写/读写必须互斥；

`共享/排它锁的潜在问题是，不能充分的并行，解决思路是数据多版本控制（MVCC）。`



##### 3.2 自增锁（Auto-inc Locks）

在InnoDB存储引擎的内存结构中，对每个含有自增长值的表都有一个自增长计数器（auto-increment counter）。对含有自增长的计数器的表进行插入操作时，这个计数器会被初始化。插入操作会依据这个自增长的计数器值加1赋予自增长列。这个实现的方式称为AUTO-INC locking。

通过以下命令查看表的自增长计数器的值：

```mysql
select max(auto_inc_col) from <table_name> for update
```



`自增锁是一种特殊的表级别锁（table-level lock）`，专门针对事务插入`AUTO_INCREMENT`类型的列。最简单的情况，如果一个事务正在往表中插入记录，所有其他事务的插入必须等待，以便第一个事务插入的行，是连续的主键值。（**注：自增锁不是在一个事务完成后才释放，而是在完成对自增长值插入的SQL语句后立即释放**）

*在InnoDB存储引擎中，自增长值的列必须是索引，同时必须是索引的第一个列。*



InnoDB提供了`innodb_autoinc_lock_mode`配置，可以调节与改变该锁的模式与行为。
一共提供了三种模式可供选择：  

- 0：`traditonal` （每次都会产生表锁）  
- 1：`consecutive` （默认．会产生一个轻量锁，simple insert会获得批量的锁，保证连续插入）  
- 2：`interleaved` （不会锁表，来一个处理一个，并发最高）  



mysql不支持直接设置该变量，会抛出以下异常：

```
mysql> set innodb_autoinc_lock_mode = 1;
ERROR 1238 (HY000): Variable 'innodb_autoinc_lock_mode' is a read only variable
```



修改方式：修改mysqld配置文件的方式（Ubuntu16.04版本中，该配置文件路径为：`/etc/mysql/mysql.conf.d/mysqld.cnf`）：    
在[mysqld]下添加配置项，例如：  

```tex /etc/mysql/mysql.conf.d/mysqld.cnf
innodb_autoinc_lock_mode = 0
```
然后重启mysql服务

```
sudo service mysql restart
```



示例：

```mysql
MySQL，InnoDB，默认的隔离级别(RR)，假设有数据表：
t_id_incr(id AUTO_INCREMENT, name);
 
数据表中有数据：
1, shenjian
2, zhangsan
3, lisi

按以下顺序执行，对于事务Ｂ会阻塞吗？

A:set autocommit = 0;
A:start transaction;
A:insert into t_id_incr(name) values('ooo');
	B:set autocommit = 0;
	B:start transaction;
	B:insert into t_id_incr(name) values('xxx');
A:insert into t_id_incr(name) values('xoo');
A:select * from t_id_incr;
```
<u>该示例，将innodb_autoinc_lock_mode修改为0,1,2任何一个值，事务B依旧均没有阻塞？？？？这是正常的吗？？？？</u>



##### 3.3 意向锁（Intention Locks）

InnoDB支持多粒度锁(multiple granularity locking)，它允许行级锁与表级锁共存．因此InnoDB支持一个额外的锁方式，即意向锁（Intention Lock）．



意向锁是将锁定的对象分为多个层次，意味着事务希望在更细的粒度上进行加锁．例如：`需要对页上的记录ｒ进行上Ｘ锁，那么分别需要对数据库Ａ，表，页上意向锁IX，最后低记录ｒ上Ｘ锁．所起中任何一个部分导致等待，那么该操作需要等待粗粒度锁的完成．`



实际上，InnoDB支持的意向锁设计简练，即为表锁．是为了在一个事务中揭示下一行将被请求的锁类型． 

InnoDB意向锁的特点：
- ①、意向锁是一个表级别的锁（table-level locking）;

- ②、意向锁又分为：
    - **意向共享锁**（Intention share lock, IS）, 它预示着事务有意向对表中某些行加共享S锁。
    - **意向排它锁**（Intention exclusive lock, IX）,它预示着，事务有意向对表中某些行加排它锁。
    例如：
    ```mysql
    select ... lock in share mode // 要设置IS锁；
    select ... for update // 要设置IX锁
    ```
    
- 意向锁协议
    - 事务要获得某些行的S锁，必须获得表的IS锁；
    - 事务要获得某些行的X锁，必须获得表的IX锁；

- `由于意向锁仅仅表明意向，它其实是比较弱的锁，意向锁之间并不相互互斥，而是可以并行`。

- `但是意向锁会和共享锁/排他锁互斥`， 其兼容互斥表如下：
  
    |      | IS   | IX   | S    | X    |
    | ---- | ---- | ---- | ---- | ---- |
    | IS   | 兼容 | 兼容 | 兼容 | 互斥 |
    | IX   | 兼容 | 兼容 | 互斥 | 互斥 |
    | S    | 兼容 | 互斥 | 兼容 | 互斥 |
    | X    | 互斥 | 互斥 | 互斥 | 互斥 |



##### 3.4 插入意向锁（Insert Intention Locks）

插入意向锁是间隙锁（Gap Locks）的一种（实施在索引上的），他是专门针对insert操作的。

特点：
多个事务，在同一索引，同一个范围区间插入记录时，如果插入的位置不冲突，不会阻塞彼此。
（官网原文：Insert Intention Lock signals the intent to insert in such a way that multiple transactions inserting into the same index gap need not wait for each other if they are not inserting at the same position within the gap.）

示例：

```
在MySQL，InnoDB，RR下：
t(id unique PK, name);
 
数据表中有数据：
10, shenjian
20, zhangsan
30, lisi
 
事务A先执行，在10与20两条记录中插入了一行，还未提交：
insert into t values(11, xxx);
 
事务B后执行，也在10与20两条记录中插入了一行：
insert into t values(12, ooo);
 
(1)会使用什么锁？
(2)事务B会不会被阻塞呢？
```
虽然事务隔离级别是RR，虽然是同一个索引，虽然是同一个区间，但插入的记录并不冲突，故这里：
- 使用的是插入意向锁
- 并不会阻塞事务B



##### 3.5 记录锁（Record Locks）

记录锁，它封锁索引记录。  
例如：

```
select * from t where id=1 for update;
```
该示例会在id=1的索引记录上加锁，以阻止其他事务插入、更新、删除id=1的这一行。

注：以下语句为快照读（SnapShot Read）
```
select * from t where id=1;
```



##### 3.6 间隙锁（Gap Locks）

间隙锁，它封锁索引记录中的间隔（*不包含记录本身*），或者第一条索引记录之前的范围，又或者最后一条索引记录之后的范围。

```
例子，InnoDB，RR隔离级别下：
t(id PK, name KEY, sex, flag);
 
表中有四条记录：
1, shenjian, m, A
3, zhangsan, m, A
5, lisi, m, A
9, wangwu, f, B
 
这个SQL语句
select * from t 
    where id between 8 and 15 
    for update;
会封锁区间，以阻止其他事务id=10的记录插入
```

<u>间隙锁的作用是为了阻止多个事务将记录插入到同一个范围内，而这会导致Phantom Problem（幻象问题，MySQL定义的是不可重复读）问题的产生．</u>

`显式关闭间隙锁的方法`：

- 将事务的隔离级别设置为READ COMMITTED.
- 将参数innodb_locks_unsafe_for_binlog设置为１．



`删除不存在的记录，获取到的是共享间隙锁．`



##### 3.7 临键锁（Next-Key Locks）

临键锁，`是记录锁与间隙锁的组合，它的封锁范围，既包含索引记录，又包含索引区间。`InnoDB存储引擎默认**对于行的查询都是采用临键锁的算法**．
（临键锁会封锁索引记录本身，以及索引记录之前的区间）。

`如果一个Session（会话）占有了索引记录R的共享/排他锁，其他会话不能立刻在R之前的区间插入新的索引记录。`
（官方文档原文：If one session has a shared or exclusive lock on record R in an index, another session cannot insert a new index record in the gap immediately before R in the index order.）



例如：一个索引有10, 11, 13, 20这四个值，那么该索引可能被Next-Key Locking的区间为：

```
(-infinity, 10]
(10, 11]
(11, 13]
(13, 20]
(20, +infinity]
```



其实除了Next-Key Locking外，还有一种`Previous-key Locking`技术，那么可锁定的区间是：

```
(-infinity, 10)
[10, 11)
[11, 13)
[13, 20)
[20, +infinity)
```



- 当查询的索引含有唯一属性时

  - 等值查询的情况：InnoDB存储引擎会对Next-Key Lock进行优化，将其降级为Record Lock，即仅锁住索引本身．
  - 范围查询的情况：InnoDB存储引擎会使用Next-Key Lock．

  

  示例：

```mysql
１．创建测试表
create table t (a int primary key);

２．写入测试数据
insert into t select 1;
insert into t select 2;
insert into t select 5;

３．等值查询的情况．
A: set autocommit=0;
A: begin;
A: select * from t where a=5 for update;
	B: set autocommit=0;
	B: insert into t select 4; // 成功，执行时并未发生阻塞．
A: commit;
	B: commit;

４．进行范围查询的情况．
A: set autocommit=0;
A: begin;
A: select * from t where a>2 for update; // 范围查询时，对(2,+infinity)范围添加了Ｘ锁．
	B: set autocommit=0;
	B: insert into t select 4; // 失败，执行是发生阻塞．
A: commit;
	B: commit;
status
```

- `当查询的索引为辅助索引，InnoDB存储引擎将会对辅助索引使用Next-Key Locking技术加锁，并且对辅助索引的下一个键值加上Gap Locks，同时对聚集索引，添加Record Locking．`

```mysql
１．创建测试表
create table z (a int, b int, primary key(a), key(b));

２．写入测试数据
insert into z select 1, 1;
insert into z select 3, 1;
insert into z select 5, 3;
insert into z select 7, 6;
insert into z select 10, 8;

３．两个事务按以下顺序执行：
A: set autocommit=0;
A: begin;
A: select * from z where b = 3 for update; // 对于辅助索引：(1,3]添加临键锁；(3,6)添加间隙锁；对于聚簇索引a=5添加记录锁．
	B: set autocommit=0;
	B: select * from z where a = 5 lock in share mode; // 阻塞．
	B: insert into z select 4, 2;　// 阻塞，ａ=4没问题，但是b=2被临键锁锁定．
	B: insert into z select 6, 5;　// 阻塞，a=6没问题，但是ｂ=5被间隙锁锁定．
```

临键锁的目的，也是为了避免幻读，如果把事务的隔离级别降为RC，临键锁则也会失效。



3.7.1 Phantom Problem(幻象问题，不可重复读)

`Phantom Problem(幻读)是指在同一事务下，连续执行两次同样的SQL语句可能导致不同的结果，第二次的SQL语句可能会返回之前不存在的行．`

在默认的事务隔离级别（READ REPEATABLE）下，InnoDB采用`Next-Key Locking机制`来避免Phantom Problem.



注意：<u>MySQL官方文档中，将`不可重复读`的问题定义为Phantom Problem，即幻象问题．</u>



#### 4 InnoDB一致性读

InnoDB提供了两种一致性读，分别是：

- 一致性非锁定读（Consistent nonlocking read）
- 一致性锁定读（Consistent locking read）



##### 4.1 一致性非锁定读

一致性非锁定读是指InnoDB存储引擎通过多版本控制的方式来读取当前执行时间数据库中的行数据．若读取的行正在执行DELETE或UPDATE操作，这时`读取操作不会因此去等待行上的锁释放．而是去读取行的一个快照`．

如下图所示：

<img src="https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/MySQL-并发控制/InnoDB存储引擎非锁定的一致性读.png" alt="InnoDB存储引擎非锁定的一致性读" style="zoom:80%;" />  



`什么是快照数据呢？`

快照数据是指行的之前版本的数据，该`实现是通过undo段来完成的`．而undo段用来在事务中回滚数据，因此，快照数据本身是没有任何开销的．此外，读快照数据是不需要上锁的．



需要注意的是，并不是所有的隔离级别都是采用一致性非锁定读．实际上`只有在READ COMMITTED和REPEATABLE READ（默认隔离级别）隔离级别下，ＩnnoDB采用一致性非锁定读`．但是他们的快照数据定义却不相同．

- READ COMMITTED：总是读取最新的一份快照数据．
- REPEATABLE READ：总是读取最老的（事务最开始的那份）一份快照数据．



##### 4.2 一致性锁定读

对于默认的隔离级别REPEATABLE READ模式下，如果想要显示的通过加锁的方式保证数据的一致性，InnoDB提供了一种加锁操作．如下：

```mysql
select ... for update // 对读取行记录加一个Ｘ（排他）锁
select ... lock in share mode　// 对读取行记录加一个Ｓ（共享）锁
```

注：使用以上两种方式显示加锁，需要加上`begin`, `start transaction`或`set autocommit=0`的操作．



<u>对于一致性非锁定读，使用`select ... for update`方式无效．仍然以快照读的方式运行．</u>



#### 5 死锁

##### 5.1 死锁的概念

死锁是`指来两个或者两个以上的事务在执行过程中，因争夺锁资源而造成的一种互相等待的现象．`



`解决死锁的办法`：

- 超时：该方法是最简单的方法，即当两个事务相互等待时，当其中一个等待时间超过设置的某一阈值，将其中一个事务进行回滚，另一个等待的事务继续进行．
  - 问题：仅通过超时回滚的方式，若超时的事务权重占比较大（如事务操作更新了很多行，占用较多的undo log），回滚该事务可能相对另一个事务占用的时间更多．
- 死锁检测：当前数据库普遍采用`Wait-For Gragh（等待图）的方式来进行死锁检测`．InnoDB即采用该方式．



###### 5.1.1 Wait-For Gragh（等待图）

Wait-For Gragh要求数据库通过使用以下两种信息链表构造出一张图，若图中存在回路，就表示存在死锁，因此资源间相互发生等待．

- `锁的信息链表`
- `事务等待链表`

*注：Wait-For Gragh中箭头（T1->T2）指向表示事务T1等待事务T2所占用的资源．*



Wait-For Gragh是一种较为主动的死锁检测机制，在每个事务请求锁并发生等待时会判断是否存在回路，若存在死锁，通常来说InnoDB存储引擎选择回滚undo量最小的事务．



`示例`：当前事务和锁的状态如下图所示．

<img src="https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/MySQL-并发控制/事务和锁等待信息.png" alt="事务和锁等待信息" style="zoom:80%;" />

其wait-for gragh如下，可见（t1, t2）存在回路，因此存在死锁．

<img src="https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/MySQL-并发控制/wait-for-gragh.png" alt="wait-for-gragh" style="zoom:67%;" />



##### 5.2 死锁的示例

后续示例均在以下表中操作

```mysql
# 1. 建表
CREATE TABLE `t` (
  `a` int(11) NOT NULL,
  PRIMARY KEY (`a`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1

# 2. 构建测试数据
+-----+
|  a  |
+-----+
|  1  |
|  2  |
|  4  |
|  5  |
|  11 |
+-----+
```



###### 5.2.1 AB-BA情况

AB-BA情况，是最经典的死锁情况，即Ａ等待Ｂ，Ｂ在等待Ａ．

```mysql
A: begin;
A: sselect * from t where a=1 for update;
	B: begin;
	B: select * from t where a=2 for update;
A: select * from t where a=2 for update; # 阻塞
	B: select * from t where a=1 for update; # 发生死锁.
	ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
A: 获取锁，返回结果．
```

发现死锁后，InnoDB存储引擎会马上回滚一个事务，另一个阻塞事务将继续进行．



###### 5.2.2 共享锁/排他锁死锁

当前事务持有待插入记录的下一个记录的Ｘ锁，但在等待队列中存在一个Ｓ锁的请求，则可能发生死锁．

```　mysql
A: begin;
	B: begin;
A: select * from t where a=4 for update; # 对ａ=4持有一个排他锁．
	B: select * from t where a<=4 lock in share mode; # 等待对a<=4的记录获取一个Ｓ锁．
A: insert into t values(3);　# 发生死锁，回滚undo log记录大的事务．
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
	B: 获得锁，返回结果．
```

```tex
+-----+     +-----+  
| a=4 |     | a=3 |
+-----+     +-----+
| A:X |     | B:S |
| B:S |
+-----+
 
     Ｂ等待Ａ释放a=4的排他锁
A<--------------------------B
 -------------------------->
 	 A等待B释放a=3的共享锁
```

注：以上示例与AB-BA死锁处理方式不同，此处选择回滚undo log记录大的事务．(为什么呢？？？？？？？？)



###### 5.2.3 并发间隙锁的死锁

```MYSQL
# 1. 准备测试数据，准备一个大的间隙的数据．
insert into t values(11);

# 2. 开始测试．
A: begin;
A: delete from t where a=7; #　删除一个不存在的记录，获取(5, 11)共享间隙锁．
	B: begin;
	B: delete from t where a=8; #　删除一个不存在的记录，获取(5, 11)共享间隙锁．
A: insert into t values(9); # 插入数据，希望获得（5, 11)的排他间隙锁，于是会阻塞．
	B: insert into t values(10);　# 插入数据，也希望获得（5, 11)的排他间隙锁，此时出现死锁．
	ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
A: 由于事务Ｂ出现死锁，回滚，因此Ａ插入成功．
```



通过`show engine innodb status`命令查看死锁情况，可知，该死锁确实是由于locks gap导致的．



---

#### 参考文档

①、[InnoDB，select为啥会阻塞insert？](https://mp.weixin.qq.com/s/y_f2qrZvZe_F4_HPnwVjOw) 
②、[InnoDB并发插入，居然使用意向锁？](https://mp.weixin.qq.com/s/iViStnwUyypwTkQHWDIR_w) 
③、[插入InnoDB自增列，居然是表锁？](https://mp.weixin.qq.com/s/kOMSD_Satu9v9ciZVvNw8Q) 
④、[InnoDB并发如此高，原因竟然在这？](https://mp.weixin.qq.com/s/R3yuitWpHHGWxsUcE0qIRQ)  
⑤、MySQL技术内幕+InnoDB存储引擎
⑥、高性能ＭySQL 第三版

