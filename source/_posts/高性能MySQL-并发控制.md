---
title: 高性能MySQL-并发控制
date: 2020-09-10 16:54:48
tags: ["MySQL","Note"]
categories: ["MySQL", "Note", "高性能MySQL"]
---

无论何时，只要有多个查询需要在同一时刻修改数据，都会产生并发控制的问题．通常并发控制是采用`锁的方式`来防止数据不一致．

<!--more-->

#### 1、读写锁

在处理并发读和并发写时，可以通过实现一个由两种类型的锁组成的锁系统来解决，分别是：  
- ①、共享锁（shared lock）——读锁， 多个客户在同一时刻读取同一个资源，而互不干扰。
- ②、排它锁（exclusive lock）——写锁， 写锁会阻塞其他的写锁和读锁。

#### 2、锁粒度
一种提高共享资源并发性的方式就是让锁定对象更具有选择性．尽量只锁定需要修改的部分数据，而不是所有的资源．更理想的方式是只对对修改的数据片进行精确锁定．`任何时候，在给定的资源上，锁定的数据量越少，并发度越高．`

问题是，加锁需要消耗资源．因此需要寻求一种策略，`在锁的开销和数据安全性之间做平衡，这就是锁策略`．



Mysql提供了多钟锁策略，每种Mysql`存储引擎都可以实现自己的锁策略和锁粒度`。
锁定的资源越少，系统并发程度越高，但加锁消耗的资源越大（获得锁，检查锁，释放锁等操作）。

##### 2.1、表锁（table lock）
表锁是Mysql中最基本的锁策略，并且是开销最小的锁策略。它将会锁定整张表。其写锁比读锁有更高的优先级，因此一个写锁请求可能会被插入到读锁请求队列的前面．

某些情况下，Mysql自身会使用表锁实现目的，并忽略存储引擎的锁机制，例如，ALTER TABLE之类的语句使用的表锁。

##### 2.2、行级锁（row lock）
行级锁可以最大程度地支持并发处理，但同时也带来了最大锁开销。行级锁在存储引擎层实现的。

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
在InnoDB里实现了标准的行级锁(row-level locking)，共享/排它锁：  
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

自增锁是一种特殊的表级别锁（table-level lock），专门针对事务插入`AUTO_INCREMENT`类型的列。最简单的情况，如果一个事务正在往表中插入记录，所有其他事务的插入必须等待，以便第一个事务插入的行，是连续的主键值。

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
t(id AUTO_INCREMENT, name);
 
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
该示例，将innodb_autoinc_lock_mode修改为0后，事务B依旧没有阻塞

##### 3.3 意向锁（Intention Locks）
InnoDB支持多粒度锁(multiple granularity locking)，它允许行级锁与表级锁共存，实际应用中，InnoDB使用的是意向锁。 

意向锁指的是：未来的某个时刻，事务可能要加共享/排他锁了，先提前声明一个意向。

意向锁的特点：
- ①、意向锁是一个表级别的锁（table-level locking）;
- ②、意向锁又分为：
    - **意向共享锁**（intention share lock, IS）, 它预示着事务有意向对表中某些行加共享S锁。
    - **意向排它锁**（intention exclusive lock, IX）,它预示着，事务有意向对表中某些行加排它锁。
    例如：
    ```
    select ... lock in share mode,要设置IS锁；
    select ... for update, 要设置IX锁
    ```
- 意向锁协议
    - 事务要获得某些行的S锁，必须获得表的IS锁；
    - 事务要获得某些行的X锁，必须获得表的IX锁；

- ==由于意向锁仅仅表明意向，它其实是比较弱的锁，意向锁之间并不相互互斥，而是**可以并行**==。
- ==但是意向锁会和共享锁/排他锁互斥==， 其兼容互斥表如下：
    |      | S    | X    |
    | ---- | ---- | ---- |
    | IS   | 兼容 | 互斥 |
    | IX   | 互斥 | 互斥 |

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
间隙锁，它封锁索引记录中的间隔，或者第一条索引记录之前的范围，又或者最后一条索引记录之后的范围。

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

间隙锁的目的，是为了防止其他事务在间隔中插入数据，以导致“不可重复读”。

若将事务的隔离级别降为RC，间隙锁则自动失效。

##### 3.7 临键锁（Next-Key Locks）
临键锁，是记录锁与间隙锁的组合，它的封锁范围，既包含索引记录，又包含索引区间。
（临键锁会封锁索引记录本省，以及索引记录之前的区间）。

如果一个Session（会话）占有了索引记录R的共享/排他锁，其他会话不能立刻在R之前的区间插入新的索引记录。
（官方文档原文：If one session has a shared or exclusive lock on record R in an index, another session cannot insert a new index record in the gap immediately before R in the index order.）

```
例子：InnoDB，RR：
t(id PK, name KEY, sex, flag);
 
表中有四条记录：
1, shenjian, m, A
3, zhangsan, m, A
5, lisi, m, A
9, wangwu, f, B
 
PK上潜在的临键锁为：
(-infinity, 1]
(1, 3]
(3, 5]
(5, 9]
(9, +infinity]
```
临键锁的目的，也是为了避免幻读，如果把事务的隔离级别降为RC，临键锁则也会失效。

---



参考文档：  
1 [InnoDB，select为啥会阻塞insert？](https://mp.weixin.qq.com/s/y_f2qrZvZe_F4_HPnwVjOw) 
2 [InnoDB并发插入，居然使用意向锁？](https://mp.weixin.qq.com/s/iViStnwUyypwTkQHWDIR_w) 
3 [插入InnoDB自增列，居然是表锁？](https://mp.weixin.qq.com/s/kOMSD_Satu9v9ciZVvNw8Q)  

4 [InnoDB并发如此高，原因竟然在这？](https://mp.weixin.qq.com/s/R3yuitWpHHGWxsUcE0qIRQ)  