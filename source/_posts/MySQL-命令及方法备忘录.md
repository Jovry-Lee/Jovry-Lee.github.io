---

title: MySQL-命令/方法备忘录
date: 2020-09-11 11:52:09
tags: ["MySQL","InnoDB"]
categories: ["MySQL","InnoDB"]
---



记录一些常用或者不常用的命令或方法，方便后续直接查看．．．

<!--more-->

### 1 简单监控当前事务及分析锁问题

从InnoDB1.0开始，在`INFORMATION_SCHEMA`架构下添加了表INNODB_TRX，INNODB_LOCKS, INNODB_LOCK_WAITS．`通过这三张表可以简单的监控当前事务并分析可能存在的锁问题`．

示例：

```mysql
表结构：
mysql> explain t_id_incr;
+-------+------------------+------+-----+---------+----------------+
| Field | Type             | Null | Key | Default | Extra          |
+-------+------------------+------+-----+---------+----------------+
| id    | int(10) unsigned | NO   | MUL | NULL    | auto_increment |
| name  | varchar(10)      | NO   |     |         |                |
+-------+------------------+------+-----+---------+----------------+
2 rows in set (0.00 sec)

表数据如下：
mysql> select * from t_id_incr;
+----+----------+
| id | name     |
+----+----------+
|  1 | seven    |
|  2 | zhangsan |
|  3 | lisi     |
+----+----------+


按以下顺序执行，产生锁等待
Ａ：start transaction;
Ａ：update t_id_idcr set name='seven1' where id = 1;
	Ｂ：start transaction;
	Ｂ：update t_id_incr set name='zhangsan1' where id = 2;
Ａ：update t_id_incr set name='zhangsan2' where id = 2;
	
此时由于事务Ｂ正在修改id=2的行，因此Ａ需等待事务Ｂ释放id=2的行锁．
```



#### 1.1 INNODB_TRX表

INNODB_TRX表提供了当前INNODB引擎内每个事务的信息，包括事务是否在锁等待，正在执行的语句，隔离级别等等．

详细信息可查看：[INNODB_TRX](https://dev.mysql.com/doc/refman/8.0/en/information-schema-innodb-trx-table.html)



查看innodb_trx表信息：

```mysql
mysql> select * from information_schema.INNODB_TRX\G;
*************************** 1. row ***************************
                    trx_id: 101153
                 trx_state: RUNNING
               trx_started: 2020-09-11 16:43:15
     trx_requested_lock_id: NULL
          trx_wait_started: NULL
                trx_weight: 5
       trx_mysql_thread_id: 6
                 trx_query: select * from information_schema.INNODB_TRX
       trx_operation_state: NULL
         trx_tables_in_use: 0
         trx_tables_locked: 1
          trx_lock_structs: 4
     trx_lock_memory_bytes: 1136
           trx_rows_locked: 3
         trx_rows_modified: 1
   trx_concurrency_tickets: 0
       trx_isolation_level: REPEATABLE READ
         trx_unique_checks: 1
    trx_foreign_key_checks: 1
trx_last_foreign_key_error: NULL
 trx_adaptive_hash_latched: 0
 trx_adaptive_hash_timeout: 0
          trx_is_read_only: 0
trx_autocommit_non_locking: 0
*************************** 2. row ***************************
                    trx_id: 101152
                 trx_state: LOCK WAIT
               trx_started: 2020-09-11 16:43:05
     trx_requested_lock_id: 101152:444:4:3
          trx_wait_started: 2020-09-11 16:43:22
                trx_weight: 6
       trx_mysql_thread_id: 5
                 trx_query: update t_id_incr set name='zhangsan2' where id = 2
       trx_operation_state: starting index read
         trx_tables_in_use: 1
         trx_tables_locked: 1
          trx_lock_structs: 5
     trx_lock_memory_bytes: 1136
           trx_rows_locked: 4
         trx_rows_modified: 1
   trx_concurrency_tickets: 0
       trx_isolation_level: REPEATABLE READ
         trx_unique_checks: 1
    trx_foreign_key_checks: 1
trx_last_foreign_key_error: NULL
 trx_adaptive_hash_latched: 0
 trx_adaptive_hash_timeout: 0
          trx_is_read_only: 0
trx_autocommit_non_locking: 0
2 rows in set (0.00 sec)
```





#### 1.2 INNODB_LOCKS表

INNODB_LOCKS表提供关于InnoDB事务已请求但尚未获取的每个锁的信息，以及事务持有的阻止另一个事务的每个锁的信息．（ MySQL 8.0.1起不推荐使用该表，并删除了该表。改用性能架构[`data_locks`](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-data-locks-table.html) 表）

详细信息可查看：[INNODB_LOCKS](https://dev.mysql.com/doc/refman/8.0/en/information-schema-innodb-locks-table.html)

查看innodb_locks表：

```mysql
mysql> select * from information_schema.innodb_locks\G;
*************************** 1. row ***************************
    lock_id: 101152:444:4:3
lock_trx_id: 101152
  lock_mode: X
  lock_type: RECORD
 lock_table: `seven`.`t_id_incr`
 lock_index: idx_id
 lock_space: 444
  lock_page: 4
   lock_rec: 3
  lock_data: 2, 0x000000000301
*************************** 2. row ***************************
    lock_id: 101153:444:4:3
lock_trx_id: 101153
  lock_mode: X
  lock_type: RECORD
 lock_table: `seven`.`t_id_incr`
 lock_index: idx_id
 lock_space: 444
  lock_page: 4
   lock_rec: 3
  lock_data: 2, 0x000000000301
2 rows in set, 1 warning (0.02 sec)
```



#### 1.3 INNODB_LOCK_WAITS表

INNODB_LOCK_WAITS表为每个被阻止的InnoDB事务包含一个或多个行，指示它已请求的锁以及正在阻止该请求的所有锁．（MySQL 8.0.1起不推荐使用该表，并删除了该表。改用性能架构 [`data_lock_waits`](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-data-lock-waits-table.html)表。）

详细信息可查看：[INNODB_LOCK_WAITS](https://dev.mysql.com/doc/refman/8.0/en/information-schema-innodb-lock-waits-table.html)

查看innodb_lock_waits表：

```mysql
mysql> select * from information_schema.INNODB_LOCK_WAITS\G;
*************************** 1. row ***************************
requesting_trx_id: 101152
requested_lock_id: 101152:444:4:3
  blocking_trx_id: 101153
 blocking_lock_id: 101153:444:4:3
1 row in set, 1 warning (0.00 sec)
```



### INNODB_LOCK_WAITS

### 2 隔离级别

不同事务的隔离级别，InnoDB的锁实现是不一样的．支持的隔离级别包括：

- read uncommitted
- read committed
- repeatable read（默认）
- serializable



#### 2.1 查看隔离级别方法

- 查看当前会话隔离级别

```mysql
show variables like 'tx_isolation'
或
select @@tx_isolation
```

- 查看系统隔离级别

```mysql
show global variables like 'tx_isolation'
或
select @@global.tx_isolation
```



#### 2.2 设置隔离级别

- 设置当前会话隔离级别：

```mysql
set session transaction isolation level <隔离级别>';
```

- 设置系统隔离级别：

```mysql
set global transaction isolation level <隔离级别>';
```



### 3 事务自动提交

Mysql默认采用自动提交模式（即`autocommit=1`），即如果不是显式地开始一个事务，每个查询都被当做一个事务执行提交操作。

#### 3.1 查看事务自动提交模式

- 查看当前会话自动提交模式命令：

```mysql
show variables like 'autocommit'
```

- 查看数据库系统自动提交模式命令：

```mysql
show global variables like 'autocommit'
```



#### 3.2 设置事务自动提交模式

- 设置当前会话自动提交模式命令：

```mysql
set autocommit = ０　// 0表示关闭，１表示开启
```

- 设置系统自动提交模式命令：

```
set global autocommit = 0;
```



### 4 数据库线程

#### 4.1 查看数据库线程使用情况

命令：

```mysql
 show processlist
```

示例：

```mysql
mysql>  show processlist;
+----+------+-----------+-------+---------+------+----------+------------------+
| Id | User | Host      | db    | Command | Time | State    | Info             |
+----+------+-----------+-------+---------+------+----------+------------------+
|  3 | root | localhost | seven | Query   |    0 | starting | show processlist |
|  4 | root | localhost | seven | Sleep   | 1450 |          | NULL             |
+----+------+-----------+-------+---------+------+----------+------------------+
2 rows in set (0.00 sec)
```



#### 4.2 杀掉线程

命令：

```mysql
kill <id>
```

示例：

```
mysql> kill 4;
Query OK, 0 rows affected (0.00 sec)

mysql>  show processlist;
+----+------+-----------+-------+---------+------+----------+------------------+
| Id | User | Host      | db    | Command | Time | State    | Info             |
+----+------+-----------+-------+---------+------+----------+------------------+
|  3 | root | localhost | seven | Query   |    0 | starting | show processlist |
+----+------+-----------+-------+---------+------+----------+------------------+
1 row in set (0.00 sec)
```



### 5 查看表自增长计数器

在InnoDB存储引擎的内存结构中，对每个含有自增长值的表都有一个自增长计数器（auto-increment counter）。对含有自增长的计数器的表进行插入操作时，这个计数器会被初始化。

通过以下命令查看表的自增长计数器的值：

```mysql
select max(auto_inc_col) from <table_name> for update
```



### 6 显式关闭间隙锁的方法

显式关闭间隙锁的方法：

- 将事务的隔离级别设置为READ COMMITTED.
- 将参数innodb_locks_unsafe_for_binlog设置为１．



### 7 锁等待时间

在InnoDB存储引擎中，参数`innodb_lock_wait_timeout`用来控制等待的时间（默认是50秒）,该参数是动态的，可以在MySQL运行时进行调整．

- 查看锁等待时间．

```mysql
show variables like 'innodb_lock_wait_timeout';
```

- 设置锁等待时间,单位秒．

```mysql
set @@innodb_lock_wait_timeout=<超时时间>;
```



### 8 超时回滚

在InnoDB存储引擎中，参数innodb_rollback_on_timeout用来设定是否在等待超时时对进行中的事务进行回滚操作（默认是OFF，代表不回滚）．

- 查看超时回滚．

```mysql
show variables like 'innodb_rollback_on_timeout';
```

- 启用超时回滚．

  innodb_rollback_on_timeout参数为只读参数，需要更改配置文件，并重启服务才会生效．

  在mysqld.cnf文件(本机配置文件路径为：`/etc/mysql/mysql.conf.d/mysqld.cnf`)末尾添加以下内容：

  ```tex mysqld.cnf
  innodb_lock_wait_timeout=on
  ```

  











