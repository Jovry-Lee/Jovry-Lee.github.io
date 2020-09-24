---
title: 'MySQL-脏读,不可重复读,幻读'
date: 2020-09-14 17:05:55
tags: ["MySQL","事务","锁"]
categories: ["MySQL"]
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

SQL92中对不可重复读的定义如下：

> **不可重复读(nonrepeatable read)**：SQL-transaction T1 reads a row. SQL-transaction T2 then modifies or deletes that row and performs a COMMIT. If T1 then attempts to reread the row, it may receive the modified value or discover that the row has been deleted.

即，**不可重复读**是指在一个事务中多次读取同一数据行，在这个事务还没结束时，另一个事务也访问了该数据行，并做了DML操作，导致第一个事务多次读取到的数据不一样．

（注：<u>不可重复读是针对单行数据</u>）



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



MySQL官方文档中`将不可重复读的问题定义为Phantom Problem，即幻象问题`．InnoDB存储引擎的默认事务隔离级别READ REPEATABLE采用<u>Next-Key Lock算法</u>，避免了不可重复读．



### 4 幻读

SQL92中对幻读的定义如下：

> **幻读(phantom read)**：SQL-transaction T1 reads the set of rows N that satisfy some <search condition>. SQL-transaction T2 then executes SQL-statements that generate one or more rows that satisfy the <search condition> used by SQL-transaction T1. If SQL-transaction T1 then repeats the initial read with the same <search condition>, it obtains a different collection of rows.

即，**幻读**指事务T1根据一定的查询条件读取一定的数据集，在这个事务还没有结束时，另一个事务T2执行SQL生成一条或多条满足事务T1查询条件的数据行．此时事务T1再次进行相同的查询时，其结果包含不同的数据集．

（注：<u>幻读是针对一个结果集，不是单行数据</u>）



对于MySQL的默认存储引擎InnoDB在RR隔离级别下通过MVCC机制避免了幻读问题．<u>严格的来说是解决了部分幻读的问题</u>[*参考资料2*]



示例（*针对尚未解决的幻读问题*）：

```mysql
# 1. 建表
CREATE TABLE `t` (
  `a` int(11) NOT NULL,
  PRIMARY KEY (`a`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1

# 2. 数据
mysql> select * from t;
+----+
| a  |
+----+
|  1 |
|  2 |
|  3 |
+----+

# 3. 按以下顺序分别执行Ａ，Ｂ两个事务．
A: begin;
A: select * from t where a>3; # 没有大于３的结果集．
Empty set (0.00 sec)
	B: insert into t select 4;　# 另一个事务插入了一条满足a>3条件的数据．
	Query OK, 1 row affected (0.02 sec)
	Records: 1  Duplicates: 0  Warnings: 0
A: insert into t select 4; # 执行插入a=4的数据行．
ERROR 1062 (23000): Duplicate entry '4' for key 'PRIMARY'　# 对于事务Ａ在查询到a>3条件下没有结果集，写入一条数据，但是却出现了主键冲突的异常．
```


---
### 参考资料

1 MySQL技术内幕+InnoDB存储引擎第2版(7.2节)
2 [搞懂不可重复读和幻读](https://segmentfault.com/a/1190000012669504)