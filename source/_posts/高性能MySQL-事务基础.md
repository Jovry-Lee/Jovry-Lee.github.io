---
title: 高性能MySQL-事务基础
date: 2020-09-09 15:05:02
tags: ["MySQL","Note"]
categories: ["MySQL", "Note", "高性能MySQL"]
---

#### 1、简介
事务是一组原子性的SQL查询，或者说一个独立的工作单元。事务将数据库从一种一致状态转换为另一种一致状态,事务内的语句，要么全部执行成功，要么全部执行失败。

<!--more-->

#### 2、事务的ACID概念
- `A： 原子性（automicity）`
一个事务必须被视为一个不可分割的最小单元，整个事务中所有的操作要么全部提交成功，要么全部失败回滚。
```
问题: 
    对于单核单进程单个指令操作需要考虑事务的原子性吗?

理解:
    首先我们知道,在计算机中的大概分层如下,其次数据库的数据实际是存储在磁盘上,因此,针对单条指令,如insert,插入一条数
据到数据库,实际上是由应用层/CPU(针对不同的编程语言可能会不同,如C语言可编译为二进制形式直接在CPU上运行,而PHP等
语言需要通过OS)执行代码将需要的数据存储到磁盘中,此时若出现操作系统崩溃或者异常掉电的情况,则可能出现数据不一致的情况.

结论:即使单核单进程单条指令操作也是需要考虑事务的原子性.

 应用层
-------
  OS层
-------
  CPU
-------
  内存
-------
  磁盘
-------

```
- `C： 一致性（consistency）`
数据库总是从一个一致性的状态转换到另一个一致性的状态。保证数据库的完整性约束没有被破坏.
```
例如:
    在表t中有一个字段id,为唯一约束,即在表中id不能重复.假设当前表t中有一条id=1的记录,此时事务A对id=1的记录进行了删
除但尚未提交,事务B又创建了一个id=1的记录并提交,事务A回滚,若没有一致性约束,此时表t中存在两条id=1的记录,违反了id为唯一约束.

Trans A              Trans B

Start T
delete id =1
 
                      Start T
                      insert id=1
                      Commit T
Rollback T
```

- `I： 隔离性（isolation）`
    通常来说，一个事务所做的修改在最终提交前，对其他事务是不可见的。（*与隔离级别有关*)

    <u>通常通过锁来实现</u>,当前数据库系统中都提供了一种粒度锁(granular lock)的策略,允许事务仅锁住一个实体对象的子集,以此来提高事务之间的并发度.
    
- `D： 持久性（durability）`
一旦事务提交，则其所作的修改就会永久保存到数据库中。

#### 3、隔离级别
在SQL标准中定义了四种隔离级别，每一种级别都规定了一个事务中所做的修改，哪些是在事务内和事务间是可见的，哪些是不可见的。  
*（不同事物的隔离级别，实际上是一致性与并发性的一个权衡与折中。）*



##### 3.1 READ UNCOMMITTED（未提交读）（不推荐）

事务中的修改，即使没有提交，对其他事务也是可见的。
- 优点：性能上比另外三种级别好很多（并发性高）。
- 缺点：读取到未提交的数据，出现**脏读**。且缺乏其他级别的好处（`一致性最差`）。

###### 3.1.1 InnoDB的实现方式
InnoDB在此种事务隔离级别下，select语句不加锁（官方文档原文：SELECT statements are performed in a nonlocking fashion.）。



##### 3.2 READ COMMITTED（提交读）（别名：不可重复读）

事务开始直到提交之前，所做的任何修改对其他事务都是不可见的。
- 缺点：不可重复读，当两次执行同样的查询，可能会得到不一样的结果。  

###### 3.2.1 InnoDB的实现方式
- 普通读是快照读；
- 加锁的select，update，delete等语句，除了在外面约束检查以及重复键检查时会封锁区间，其他时刻都只使用记录锁；此时其他事务的插入依旧可以执行，就可能导致幻读。



##### 3.3 REPEATABLE READ（可重复读）——Mysql的默认事务隔离级别

该级别解决了`脏读`的问题，且保证同一个事务中多次读取同样的记录结果是一致的。
- 缺点：可能会出现幻读，即当某个事务在读取某个范围内的记录时，另外一个事务又在该范围插入了新的记录，就会产生幻行。
- 解决： InnoDB和XtraDB通过`多版本并发控制（MVCC, Multiversion Concurrency Control）`解决了上述缺点。

###### 3.3.1 InnoDB的实现方式
RR隔离级别为InnoDB的默认隔离级别。

①、普通的select使用快照读（snapshot read），这是一种不加锁的一致性读（Consistent Nonlocking Read），底层使用MVCC来实现。

②、加锁的select（select ... in share mode/select ... fro update）,update，delete等语句，他们的锁，依赖于他们<u>是否在唯一索引上使用了唯一的查询条件，或者范围查询条件</u>：
- 在唯一索引上使用唯一的查询条件，会使用`记录锁（record lock）`，而不会封锁记录之间的间隔，即不会使用间隙锁（gap lock）和临键锁（next-key lock）。
- 范围查询条件，会使用`间隙锁与临键锁`，锁住索引记录之间的范围，避免范围间插入记录，以避免产生幻行记录，及避免不可重读的读。



##### 3.4 SERIALIZABLE（可串行化）

强制事务串行执行，避免可重复读的幻读问题。
- 有点：一致性最好
- 缺点：该级别会在读取的每一行数据加锁，可能导致大量的超时和锁争用问题（**并发性最差**）。

###### 3.4.1 InnoDB的实现方式
这种事务的隔离级别下，所有的select语句都会被隐式的转化为select ... in share mode.  

这可能导致，如果有未提交的事务正在修改某些行，所有读取这些行的select都会被阻塞（官方文档原文：To force a plain SELECT to block if other transactions have modified the selected rows.）。

---


关于“脏读”，“不可重复读”，“幻读”示例：

```
假设有InnoDB表：
t(id PK, name);
 
表中有三条记录：
1, shenjian
2, zhangsan
3, lisi
```

- 脏读
  
  脏读指的是在不同事务下，当前事务可以读到另外事务未提交的数据．
  
```
    事务A，先执行，处于未提交的状态：
    insert into t values(4, wangwu);
 
    事务B，后执行，也未提交：
    select * from t;
   
    如果事务B能够读取到(4, wangwu)这条记录，事务A就对事务B产生了影响，这个影响叫做“读脏”，读到了未提交事务操作的记录。
```

- 不可重复读
  
    不可重复读是指在一个事务中多次读取同一数据集合，在这个事务还没结束时，另一个事务也访问了该数据集合，并做了DML操作．导致第一个事务多次读取到的数据不一样．
    
```
    事务A，先执行：
    select * from t where id=1;
 
    结果集为：
    1, shenjian
     
事务B，后执行，并且提交：
    update t set name=xxoo where id=1;
    commit;
 
    事务A，再次执行相同的查询：
    select * from t where id=1;
 
    结果集为：
    1, xxoo
     
    这次是已提交事务B对事务A产生的影响，这个影响叫做“不可重复读”，一个事务内相同的查询，得到了不同的结果。
```

- 幻读
    ```
    事务A，先执行：
    select * from t where id>3;
 
    结果集为：
    NULL
        
    事务B，后执行，并且提交：
    insert into t values(4, wangwu);
    commit;
 
    事务A，首次查询了id>3的结果为NULL，于是想插入一条为4的记录：
    insert into t values(4, xxoo);
 
    结果集为：
    Error : duplicate key!
 
    事务A的内心OS是：你TM在逗我，查了id>3为空集，insert id=4告诉我PK冲突？
 
    这次是已提交事务B对事务A产生的影响，这个影响叫做“幻读”。
    ```
    

---


#### 4、死锁
死锁是指`两个或多个事务在同一资源上相互占用，并请求锁定对方占用的资源，从而导致恶性循环的现象。`

解决方式：
- ①、数据库系统实现了各种死锁检测和死锁超时机制。
（`InnoDB处理方式：将持有最少行级排它锁的事务进行回滚`）
- ②、当查询的时间达到锁等待超时的设定后当其锁请求。



示例：

```mysql
事务A：
    start transaction;
    update stockprice set close = 45.50 where stock_id =4 and date ='2020-05-01';
    update stockprice set close = 19.80 where stock_id =3 and date = '2020-05-02';
    commit;

事务B：
    start transaction；
    update stockprice set close = 20.12 where stock_id = 3 and date = '2020-05-02';
    update stockprice set close = 47.20 where stock_id = 4 and date = '2020-05-01';
    commit;
```



#### 5、MySQL中的事务

Mysql提供了两种事务型存储引擎：`InnoDB`和`NDB Cluster`。

- 设置自动提交模式

  `Mysql默认采用自动提交模式`，即如果不是显式地开始一个事务，每个查询都被当做一个事务执行提交操作。更改自动提交的命令如下：
```
SET AUTOCOMMIT = 1;(0：启用，1：禁用)
```
- `设置隔离级别`
Mysql能识别所有的4个ANSI隔离级别，InnoDB也支持所有隔离级别。
命令示例：
```
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED
```
- Mysql中的事务是<u>由下层的存储引擎实现的，同一个事务中使用多种存储引擎是不可靠的</u>。



#### 6、多版本并发控制（MVCC）

MVCC可以被认为是行级锁的一个变种，但他在很多情况下避免了加锁操作，因此开销更低。

MVCC的实现：**通过保存数据在某个时间点的快照来实现的。**

典型的存储引擎的MVCC实现：

- 乐观并发控制（optimistic）
- 悲观并发控制（pessimistic）



##### 6.1 InnoDB的MVCC的实现

InnoDB的MVCC是通过在每行记录后面保存两个隐藏的列来实现的。这两个列一个保存了行的创建时间（系统版本号），一个保存行的过期时间（删除时间）（系统版本号）。每开始一个新的事务，系统版本号都会自动递增。事务开始时的系统版本号作为事务的版本号，用来和查询到的每行记录的版本号进行比较。



REPEATABLE READ隔离级别下，MVCC具体操作：

- `SELECT`  
InnoDB会根据以下两个条件检索：
    - a、InnoDB查找行的系统版本号小于/等于事务的系统版本号，确保事务读取的行，要么在事务开始前已经存在，要么是事务自身插入或修改过的。
    - b、行的删除版本要么未定义，要么大于当前事务版本号，确保事务读取到的行，在事务开始前未被删除。

- `INSERT`  
InnoDB为新插入的每一行保存当前系统版本号作为行版本号。
- `DELETE`  
InnoDB为删除的每一行保存当前系统版本号作为行删除标识。
- `UPDATE`  
InnoDB为新插入的每一行保存当前系统版本号作为行版本号，同时保存当前系统版本号到原来的行作为删除标识。

注：<u>MVCC只在REPEATABLE READ和READ COMMITTED两个隔离级别下工作</u>。（READ UNCOMMITTED总是读取最新的数据行，而SERIALIZABLE则会对所有读取的行都加锁）



#### 7、Mysql终端模拟并发事务

##### 7.1 几个关键的配置
要测试InnoDB的锁互斥，以及死锁，有几个配置需要提前确认：
- ①、区间锁（间隙锁，临键锁）是否关闭
- ②、事务自动提交（auto commit）是否关闭
- ③、事务的隔离级别


###### 7.1.1 区间锁是否关闭
区间锁（间隙锁，临键锁）是InnoDB特有施加在索引记录区间的锁，Mysql5.6可以手动关闭区间锁。  

控制参数：

```
innodb_locks_unsafe_for_binlog
```
该参数支持两个值：
- ON: 表示关闭区间锁，此时一致性会被破坏（所以是unsafe）
- OFF：表示开启区间锁（默认）

查询该参数的方法：

```
mysql> show variables like 'innodb_locks%';
+--------------------------------+-------+
| Variable_name                  | Value |
+--------------------------------+-------+
| innodb_locks_unsafe_for_binlog | OFF   |
+--------------------------------+-------+
1 row in set (0.03 sec)
```

修改方法：修改配置文件（/etc/mysql/mysql.conf.d/mysqld.cnf）文件，添加更改选项并重新启动服务：

```
innodb_locks_unsafe_for_binlog = 1
```

```
sudo service mysql restart
```



###### 7.1.2 事务自动提交

Mysql默认把每一个单独的SQL语句作为一个事务，自动提交。

控制参数：
```
autocommit
```

MySQL5.7默认开启
```
mysql> show variables like 'autocommit';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| autocommit    | ON    |
+---------------+-------+
1 row in set (0.00 sec)
```

设置（关闭）方式：

```
set session autocommit=0;
```



###### 7.1.3 事务的隔离级别

不同事务的隔离级别，InnoDB的锁实现是不一样的。默认为repeatable read。

控制参数：
```
tx_isolation
```
查询事务的隔离级别的方法：

```
mysql> show global variables like 'tx_isolation';
+---------------+-----------------+
| Variable_name | Value           |
+---------------+-----------------+
| tx_isolation  | REPEATABLE-READ |
+---------------+-----------------+
1 row in set (0.00 sec)
```

设置事务的隔离级别的方式：

```
set session transaction isolation level <X>';
```
其中X可取值为：
- read uncommitted
- read committed
- repeatable read
- serializable



##### 7.2 模拟并发事务前准备工作

- ①、配置准备
  要模拟并发事务，需要修改事务自动提交这个选项，每个session要改为手动提交。

```
mysql> set session autocommit=0;
Query OK, 0 rows affected (0.00 sec)

mysql> show variables like 'autocommit';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| autocommit    | OFF   |
+---------------+-------+
1 row in set (0.00 sec)
```



- ②、数据准备  
  InnoDB的行锁都是实现在索引上，实验时可以使用主键，建表时设定为innodb引擎：

  - 建表

    ```
    mysql> create table t_id_pk( id int(10) primary key) engine=innodb;
    ```

  - 准备初始数据

    ```
     mysql> insert into t_id_pk(id) values (1),(3),(10);
    ```



##### 7.3 实验

###### 7.3.1 间隙锁互斥实验
开启区间锁，RR的隔离级别下，上例会有以下四个区间：
- （-infinity， 1）
- （1， 3）
- （3， 10）
- （10， infinity）

事务A删除某个某个区间不存在的记录，获取到**共享间隙锁**，会阻止其他事务B在相应的区间插入数据，因为插入需要获取**排他间隙锁**。  
sessionA：

```mysql
mysql> set autocommit=0;
Query OK, 0 rows affected (0.00 sec)

mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> delete from t_id_pk where id = 5;
Query OK, 0 rows affected (0.05 sec)
```

sessionB：

```mysql
mysql> set autocommit=0;
Query OK, 0 rows affected (0.00 sec)

mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into t_id_pk(id)values(0);
Query OK, 1 row affected (0.00 sec)

mysql> insert into t_id_pk(id)values(2);
Query OK, 1 row affected (0.00 sec)

mysql> insert into t_id_pk(id)values(12);
Query OK, 1 row affected (0.00 sec)

mysql> insert into t_id_pk(id)values(7);

```
事务B插入的值：0，2，12都不在（3， 10）区间内，能够插入，而7在（3， 10）这个区间，会阻塞。`若事务A一直不提交，事务B就会一直等待，直到超时`，超时后会有如下提示：

```
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```


---
可以`使用以下命令查看锁的情况`：

```
show engine innodb status;
```
当前实验锁部分情况如下：

```
TRANSACTIONS
------------
Trx id counter 66634
Purge done for trx's n:o < 66634 undo n:o < 0 state: running but idle
History list length 20
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 66629, ACTIVE 52 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s), undo log entries 3
MySQL thread id 20, OS thread handle 140324098004736, query id 196 localhost root update
insert into t_id_pk(id)values(7)
------- TRX HAS BEEN WAITING 38 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 445 page no 3 n bits 80 index PRIMARY of table `seven`.`t_id_pk` trx id 66629 lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 4 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000001043f; asc      ?;;
 2: len 7; hex af00000123012a; asc     # *;;

------------------
```
可知insert into t_id_pk(id)values(7)正在等待事务A提交或回滚，这样事务B就能获得相应的锁，以继续执行。

---



###### 7.3.2 共享排它锁的死锁实验

该实验需要三个并发的session。

sessionA：
```
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into t_id_pk(id) values(7);
Query OK, 1 row affected (0.00 sec)
```

sessionB:
```
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into t_id_pk(id) values(7);
```

sessionC:
```
mysql> set autocommit=0;
Query OK, 0 rows affected (0.00 sec)

mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into t_id_pk(id) values(7);
```
此时三个事务都试图往表中插入一条为7的记录：
- A先执行，插入成功，并获取id=7的`排它锁`；
- B后执行，需要进行PK校验，故需要先获取id=7的`共享锁`，阻塞；
- C后执行，也需要进行PK校验，也要先获取id=7的`共享锁`，阻塞；

此时若sessionA执行：
```
mysql> rollback;
Query OK, 0 rows affected (0.11 sec)
```
id=7的排它锁释放，则B，C会继续进行主键校验：
- ①、B会获取到id=7共享锁，主键未互斥；
- ②、C也会获取到id=7共享锁，主键未互斥；  
B和C要想插入成功，必须获得id=7的排他锁，但由于双方都已经获取到id=7的共享锁，它们都无法获取到彼此的排他锁，死锁就出现了。



InnoDB有死锁检测机制，B和C中的一个事务会插入成功，另一个会自动放弃：
sessionB：

```
mysql> insert into t_id_pk(id) values(7);
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```
sessionC:
```
mysql> insert into t_id_pk(id) values(7);
Query OK, 1 row affected (7.40 sec)
```



###### 7.3.3 并发间隙锁的死锁

共享排它锁，在并发量插入相同记录的情况下，相应的案例比较容易分析，而并发的间隙锁死锁，是比较难定位的。
```mysql
A：set session autocommit=0;
A：start transaction;
A：delete from t where id=6;
         B：set session autocommit=0;
         B：start transaction;
         B：delete from t where id=7;
A：insert into t values(5);
         B：insert into t values(8);
```

sessionA:
```mysql
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> delete from t_id_pk  where id=6;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into t_id_pk(id)values(5);
Query OK, 1 row affected (8.73 sec)
```

sessionB:
```mysql
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> delete from t_id_pk where id=7;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into t_id_pk(id)values(8);
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```

- A执行delete后，会获得(3, 10)的共享间隙锁。
- B执行delete后，也会获得(3, 10)的共享间隙锁。
- A执行insert后，希望获得(3, 10)的排他间隙锁，于是会阻塞。
- B执行insert后，也希望获得(3, 10)的排他间隙锁，于是死锁出现。



使用show engine innodb status查看死锁的情况：

```
2019-05-14 15:05:35 0x7f9fc003d700
*** (1) TRANSACTION:
TRANSACTION 66641, ACTIVE 58 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1136, 2 row lock(s)
MySQL thread id 18, OS thread handle 140323966887680, query id 226 localhost root update
insert into t_id_pk(id)values(5)
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 445 page no 3 n bits 80 index PRIMARY of table `seven`.`t_id_pk` trx id 66641 lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 4 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000001043f; asc      ?;;
 2: len 7; hex af00000123012a; asc     # *;;

*** (2) TRANSACTION:
TRANSACTION 66642, ACTIVE 33 sec inserting
mysql tables in use 1, locked 1
3 lock struct(s), heap size 1136, 2 row lock(s)
MySQL thread id 20, OS thread handle 140324098004736, query id 227 localhost root update
insert into t_id_pk(id)values(8)
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 445 page no 3 n bits 80 index PRIMARY of table `seven`.`t_id_pk` trx id 66642 lock_mode X locks gap before rec
Record lock, heap no 4 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000001043f; asc      ?;;
 2: len 7; hex af00000123012a; asc     # *;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 445 page no 3 n bits 80 index PRIMARY of table `seven`.`t_id_pk` trx id 66642 lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 4 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000001043f; asc      ?;;
 2: len 7; hex af00000123012a; asc     # *;;

*** WE ROLL BACK TRANSACTION (2)
```

从事务锁的情况可以看出，当检测到死锁后，事务2自动回滚了。



总结：

- ①、并发事务，间隙锁可能互斥；
    - a. A删除不存在的记录，获取共享间隙锁；
    - b. B插入，必须获得排他间隙锁，故互斥；
- ②、并发插入相同的记录，可能死锁（某一个回滚）；
- ③、并发插入，可能出现间隙锁死锁（难排查）。




---
#### 参考资料
- 1. MySQL第三版
- 2. [4种事务的隔离级别，InnoDB如何巧妙实现](https://mp.weixin.qq.com/s/x_7E2R2i27Ci5O7kLQF0UA)
- 3. [超赞，InnoDB调试死锁的方法！](https://mp.weixin.qq.com/s/_36Sy0FldFRNvLRpHxfucQ)

