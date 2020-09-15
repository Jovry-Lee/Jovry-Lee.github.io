---
title: 'MySQL-脏读,不可重复读,幻读'
date: 2020-09-14 17:05:55
tags: ["MySQL","事务","锁"]
categories: ["MySQL", "事务","锁"]
---



### 1 概述

通过锁机制可以实现事务的隔离性要求，使得事务可以并发的工作．锁提高了并发，却会带来潜在问题呢，例如脏读，幻读等．

<!--more-->

后续示例均在以下user表的基础上进行操作．

```mysql
# 1.　表信息．
CREATE TABLE `users` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT COMMENT '自增主键',
  `user_name` varchar(10) NOT NULL DEFAULT '' COMMENT '用户名',
  `age` tinyint(2) NOT NULL DEFAULT '0' COMMENT '年龄',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=13 DEFAULT CHARSET=utf8mb4 COMMENT='用户信息'


# 2. 准备测试数据．
+----+-----------+-----+
| id | user_name | age |
+----+-----------+-----+
|  4 | Seven     |  18 |
|  6 | Gordon    |  20 |
| 11 | Natasha   |  23 |
| 12 | sam       |  28 |
+----+-----------+-----+
```



### 2 脏读

脏读指的是在不同事务下，当前事务可以读到另外事务未提交的数据．

`脏读发生的条件`：事务的隔离级别为READ UNCOMMITTED．

`脏读的适用场景`：在一些特殊特殊情况下，可以将事务的隔离级别设置为READ UNCOMMITTED，例如replication环境中的slave节点，并在该slave上的查询并不需要特别精确的返回值．



示例：

```mysql
A: set session transaction isolation level read uncommitted;
A: begin
A: select * from users where id>11;
+----+-----------+-----+
| id | user_name | age |
+----+-----------+-----+
| 12 | sam       |  28 |
+----+-----------+-----+
1 row in set (0.01 sec)
	B: set session transaction isolation level read uncommitted;
	B: begin
	B: insert into users(user_name,age)values('Jovry-Lee',18); # 未提交．
A: select * from users where id > 11; # 此时事务Ａ读取到事务Ｂ尚未提交的数据，属于脏读．
+----+-----------+-----+
| id | user_name | age |
+----+-----------+-----+
| 12 | sam       |  28 |
| 13 | Jovry-Lee |  18 |
+----+-----------+-----+
2 rows in set (0.00 sec)
```





### 3 不可重复读

不可重复读是指在一个事务中多次读取同一数据集合，在这个事务还没结束时，另一个事务也访问了该数据集合，并做了DML操作，导致第一个事务多次读取到的数据不一样．

`不可重复读允许发生的隔离级别`：READ COMMITTED



示例：

```mysql
A: set session transaction isolation level read committed;
A: begin
A: select * from users where id>11;
+----+-----------+-----+
| id | user_name | age |
+----+-----------+-----+
| 12 | sam       |  28 |
+----+-----------+-----+
1 row in set (0.01 sec)
	B: set session transaction isolation level read committed;
	B: begin
	B: insert into users(user_name,age)values('Jovry-Lee',18);
	B: commit;
A: select * from users where id > 11; # 此时事务Ａ两次读取到不同的结果，属于不可重复读．
+----+-----------+-----+
| id | user_name | age |
+----+-----------+-----+
| 12 | sam       |  28 |
| 13 | Jovry-Lee |  18 |
+----+-----------+-----+
2 rows in set (0.00 sec)
```



`脏读与不可重复读的区别`：脏读时读到未提交的数据，而不可重复读独到的是已经提交的数据，但时其违反了数据库事务一致性要求．

一般来说不可重复读的问题是可以接受的，因为其读到的是已经提交的数据，本身并不会带来太大的问题．因此，很多数据库的默认隔离级别是`READ COMMITTED`，如：Ｏracle, SQL Server.



MySQL官方文档中`将不可重复读的问题定义为Phantom Problem，即幻象问题`．InnoDB存储引擎的默认事务隔离级别READ REPEATABLE采用Next-Key Lock算法，避免了不可重复读．

