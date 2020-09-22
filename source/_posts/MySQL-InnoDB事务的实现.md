---
title: MySQL-InnoDB事务的实现
date: 2020-09-22 21:38:43
tags: ["MySQL","InnoDB"]
categories: ["MySQL","InnoDB"]
---

1 简介

事务隔离性由锁来实现。原子性，一致性，持久性通过数据库的`redo log`和`undo log`来实现.

<!--more-->

- **redo log(重做日志)**：<u>用于保证事务的原子性和持久性</u>。redo恢复提交事务修改的页操作，通常是物理日志,记录的是页的物理修改操作。
- **undo log(撤销日志)**：<u>用于保证事务的一致性</u>。undo回滚行记录到某个特定版本，是逻辑日志。根据每行记录进行记录。

InnoDB是事务的存储引擎，通过`Force Log at Commit机制`实现事务的持久性。即当事务提交时，必须先将该事务的所有日志写入到重做日志文件进行持久化，待事务的提交操作完成才算完成。这里的日志指重做日志，在InnoDB存储引擎中,包括redo log和undo log。



2 redo log

redo log用来保证事务的持久性。其组成包含以下两个部分:

- `redo log buffer(重做日志缓冲)`：保存在内存中,属于掉电易失的.

- `redo log file(重做日志文件)`：保存在磁盘中,属于持久的.

  

redo log基本上都是顺序写，<u>在数据库运行时不需要对redo log的文件进行读取操作</u>。

为了确保每次日志都写入`redo log file`中，在每次将`redo log buffer`写入文件后，InnoDB存储引擎都需要调用一次**fsync操作**。(重*做日志文件打开并没有使用O_DIRECT选项，因此`redo log buffer`中的数据会先写入文件系统缓存.为了保证重做日志写入磁盘,必须进行一次fsync操作.其中fsync的效率取决于磁盘的性能,因此磁盘的性能决定了事务提交的性能，即数据库的性能*)




参数`innodb_flush_log_at_trx_commit`用于控制redo log刷新到磁盘的策略。该参数可设置的为包括:
- 0：表示事务提交时不进行写入重做日志操作，仅在master thread中每1秒进行一次fsync操作。
- 1(默认)：表示事务提交时必须调用一次fsync操作。
- 2：表示事务提交时将重做日志写入重做日志文件，但仅写入文件系统的缓存中，不进行fsync操作。

注：<u>设置innodb_flush_log_at_trx_commit为0或2可以提高事务的性能，但这种方式丧失了事务的ACID特性.</u>



2.1 扩展: 二进制日志(Bin log)

MySQL数据库中还有一种二进制日志(binlog)，用来进行`POINT-IN-TIME(PIT)`的恢复及`主从复制(Replication)`环境的建立.



**Bin log与Redo log的区别?**

- 日志产生的地方不同.

  - Redo log是在InnoDB存储引擎层产生.
  - Bin log是在MySQL数据库的上层产生的，Binlog不仅是针对InnoDB存储引擎，MySQL数据库中任何存储引擎对于数据库的更改都会产生二进制日志.

- 日志记录的内容形式不同.

  - `Bin log是一种逻辑日志`，记录的是对应SQL语句.
  - `InnoDB的Redo log时物理格式的日志`，记录的时对于每个页的修改.

- 日志写入磁盘的时间点不同.如下图所示.

  ![日志写入磁盘的时间点不同](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/MySQL-InnoDB事务的实现/日志写入磁盘的时间点不同.png)

  - Bin log只在事务提交完成后进行**一次写入**，且对于每一个事务，仅包含对应事务的一个日志（`即一个事务对应一个binlog日志`）。
  - InnoDB的Redo log是在事务进行中**不断的被写入**，因此日志不是随事务提交顺序写入的。其记录的是物理操作日志，因此每个事务对应多个日志条目，且是并发的，并非在事务提交时写入，故其在文件中记录的顺序并非是事务开始的顺序。*T1、*T2、*T3表示的是事务提交时的日志。



2.2 Log block

`在InnoDB存储引擎中，重做日志都是以512字节进行存储的`。其<u>`redo log buffer`、`redo log file`都是以块（block）的方式进行保存的，称之为**重做日志块（redo log block）**，每块大小为512字节。</u>

若一个页中产生的重做日志数量大于512字节，那么需要分割为多个重做日志块进行存储，由于重做日志快的大小和磁盘扇区大小一样，都是512字节，因此重做日志的写入可以保证原子性，不需要doublewrite技术。



`重做日志块缓存的结构`如下：

![重做日志块缓存的结构](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/MySQL-InnoDB事务的实现/重做日志块缓存的结构.png)



`LOG_BLOCK_FIRST_REC_GROUP示例`：

![LOG_BLOCK_FIRST_REC_GROUP](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/MySQL-InnoDB事务的实现/LOG_BLOCK_FIRST_REC_GROUP.png)

2.3 Log group

`Log group`为**重做日志组**，<u>其中有多个redo log file</u>。虽然源码中一直有log group的镜像功能，但在ha_innobase.cc文件中禁止了该功能。**因此InnoDB存储引擎实际只有一个log group。**（Log group是一个逻辑上的概念，并没有一个实际存储的物理文件来表示log group信息。）



Redo log file中存储的是Log buffer中保存的log block，因此Redo log file也是根据块的方式进行物理存储的管理，每个块的大小也是512字节。在InnoDB存储引擎运行过程，log buffer根据一定的规则将内存中的log block刷新到磁盘。

**具体刷盘规则**如下：

- `事务提交时。`

- `当log buffer中有一半的内存空间已经被使用时。`

- `log checkpoint时`。

  

<u>写入Redo log file的方式是Append（追加），当一个文件被写满时，会接着写入下一个文件，其使用方式为：**round-robin（轮询调度算法）**。</u>

Redo log file中除了保存log buffer刷新到磁盘的log block，还保存了2KB大小的其他信息，因此对redo log file的写入并不是完全顺序的。针对log group中第一个redo log file，其前2KB的部分保存了4个512字节大小的块。（注：只有每个log group的第一个redo log file中才有2KB的其他信息。其余redo log file仅保留这些空间，但不保存。）

`Log group与Redo log file的关系图`如下：

![Log group与Redo log file的关系图](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/MySQL-InnoDB事务的实现/Log group与Redo log file的关系图.png)



2.4 重做日志格式

不同的数据库操作会有对应的重做日志格式。<u>InnoDB存储引擎的存储管理是基于页的，所以重做日志格式也是基于页的</u>。虽然有不同的重做日志格式，但他们有着通用的头部格式，如下图所示：

![重做日志格式](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/MySQL-InnoDB事务的实现/重做日志格式.png)

对于页上记录的插入和删除操作，其格式如下：

![页上记录的插入和删除操作格式](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/MySQL-InnoDB事务的实现/页上记录的插入和删除操作格式.png)

2.5 LSN（日志序列号）

在InnoDB存储引擎中，LSN占用8个字节，并且单调递增，其表示的含义有：

- 重做日志写入的总量

  - LSN记录的是重做日志的总量，其单位为字节。

- Checkpoint的位置

- 页的版本

  - 在每个页的头部，有一个FIL_PAGE_LSN，记录了该页的LSN。在页中，LSN表示该页最后刷新时LSN的大小。因为重做日志记录的是每个页的日志，因此页中的LSN用来判断页是否需要进行恢复操作。



示例：

页P1的LSN为10000，而数据库启动时，InnoDB检测到写入重做日志中的LSN为13000，并且该事务已经提交，那么数据需要进行恢复操作，将重做日志应用到P1页中。同样的对于重做日志中LSN小于P1页的LSN，不需要进行重做，因为P1页中LSN表示页已经被刷新到该位置.

通过`SHOW ENGINE INNODB STATUS`查看LSN的情况：

```mysql
mysql> show engine innodb status\g;
...
---
LOG
---
Log sequence number 11795911
Log flushed up to   11795911
Pages flushed up to 11795911
Last checkpoint at  11795902
0 pending log flushes, 0 pending chkp writes
10 log i/o's done, 0.00 log i/o's/second
...
1 row in set (0.00 sec)
```

`Log sequence number`：表示当前LSN

`Log flushed up to`：表示刷新到重做日志文件的LSN.

`Last checkpoint at`：表示刷新到磁盘的LSN

其中以上三个参数可能是不同的，因为在一个事务中从日志缓冲刷新到重做日志文件并不只是在事务提交时发生，每秒都会有从日志缓冲刷新到重做日志文件的动作。



2.6 恢复

InnoDB存储引擎在启动时不管上次数据库运行时是否正常关闭，都会尝试进行恢复操作。由于checkpoint表示已经刷新到磁盘上的LSN，因此在恢复过程中仅需恢复checkpoint开始的日志部分。

示例：

![恢复的例子](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/MySQL-InnoDB事务的实现/恢复的例子.png)

3 undo log

3.1 基本概念

**undo log用来帮助事务回滚及MVCC功能.**

  	在对数据库进行修改时，InnoDB存储引擎不但会产生redo，还会产生一定量的undo。当用户之心事务或语句由于某某种原因是败了，又或者用户用一条ROLLBACK语句请求回滚，就可以利用这些undo信息将数据回滚到修改之前的样子。

redo存放在重做日志文件中，而undo存放在数据库内部的一个特殊段（segment）中，这个段称为undo段（undo segment）。undo段位于共享表空间内。

undo是逻辑日志，因此只是将数据库逻辑地恢复到原来的样子。当InnoDB存储引擎回滚时，它实际上做的是与先前相反的工作。对于每个INSERT,INnoDB存储引擎会完成一个DELETE;对于每个UPDATE，InnoDB存储引擎会执行一个相反的UPDATE,将修改前的行放回去。

除了回滚操作，undo的另一个作用是MVCC，即在InnoDB存储引擎中的MVCC的是显示通过undo来完成的。当用户读取一行记录时，若该记录已经被其他事务占用，当前事务可以通过undo读取之前的行版本信息，以此实现非锁定读取。

注：undo log会产生redo log，也就是undo log的产生会伴随着redo log的产生，这是因为undo log也需要持久性的保护。

3.2 undo存储管理

InnoDB存储引擎对undo的管理同样采用段的方式。InnoDB存储引擎有rollback segment，每个回滚段中记录了1024个undo log segment，而在每个undo log segment段中进行undo页的申请。共享表空间偏移量为5的页（0,5）记录了所有rollback segment header所在的页，这个页的类型为FIL_PAGE_TYPE_SYS。

从InnoDB1.2版本开始，可通过参数对rollback segment做进一步设置。其中包含：

- innodb_undo_directory：用于设置rollback segment文件所在路径，默认值为‘.’,表示当前InnoDB存储引擎的目录。
- innodb_undo_logs：用于设置rollback segment的个数，默认值为128.
- innodb_undo_tabklespaces：用于设置构成rollback segment文件的数量，这样rollback segment可以较为平均地分不到多个文件中。设置该参数后，会在路径innodb_undo_directory看到undo为前缀的文件，该文件就代表rollback segment文件。如下所示：

mysql> show variables like 'innodb_undo%'; +--------------------------+-------+ | Variable_name            | Value | +--------------------------+-------+ | innodb_undo_directory    | ./    | | innodb_undo_log_truncate | OFF   | | innodb_undo_logs         | 128   | | innodb_undo_tablespaces  | 0     | +--------------------------+-------+ 4 rows in set (0.03 sec) mysql> show variables like 'datadir'; +---------------+-----------------+ | Variable_name | Value           | +---------------+-----------------+ | datadir       | /var/lib/mysql/ | +---------------+-----------------+ 1 row in set (0.00 sec) mysql> system ls -lh /var/lib/mysql/undo*; ls: error initializing month strings ls: cannot access '/var/lib/mysql/undo*': Permission denied

注：事务在undo log segment分配页并写入undo log的这个过程同样需要写入重做日志。

Q1：当事务提交时，InnoDB存储引擎会做什么操作？

当事务提交时，InooDB会做以下两种操作：

- ①、将undo log放入链表中，以供之后的purge操作
- ②、判断undo log所在的页是否可以重用，若可以分配给下个事务使用。

Q2：为什么事务提交后并不能马上删除undo log及undo log所在的页？

因为此时可能还有其他事务需要通过undo log来的到行记录之前的版本。当事务提交时将undo log放入一个链表中，是否可以最终删除undo log及undo log所在页由purge线程来判断。

由于事务提交时，可能并不能马上释放页，在大量更新删除操作的情况下会非常浪费存储空间。因此InnoDB存储引擎在设计中对undo页也可以重用。其具体重用方式如下：

- ①、当事务提交时，首先将undo log放入链表中。
- ②、判断undo页的使用空间是否小于3/4，若是，则表示undo页可以被重用，之后新的undo log记录可能存放在当前undo log的后面。

注：存放undo log链表是以记录进行组织的，而undo页可能存放着不同事务的undo log，因此purge操作需要设计磁盘的离散读取操作，是一个比较缓慢的过程。

可以通过SHOW ENGINE INNODB STATUS来查看链表中undo log的数量，如：

mysql> show engine innodb status\g; ... ------------ TRANSACTIONS ------------ Trx id counter 1795 Purge done for trx's n:o < 0 undo n:o < 0 state: running but idle History list length 0 LIST OF TRANSACTIONS FOR EACH SESSION: ---TRANSACTION 422058470606688, not started 0 lock struct(s), heap size 1136, 0 row lock(s) ...

其中History list length表示undo log的数量，这里为0,。purge操作会减少该值，然而由于undo log所在的页可以被重用，因此即使操作发生，History list length的值也可能不为0。

3.3 undo log格式

在InnoDB存储引擎中，undo log分为：

- ①、insert undo log 
- ②、update undo log

3.3.1 Insert undo log

Insert undo log是指在insert操作中产生的undo log。因为insert操作记录，只针对事务本身可见，对其他事务不可见（这是事务隔离性的要求），所以undo log可以在事务提交后直接删除。不需要进行purge操作。**Insert undo log的格式**如下：

![Insert-undo-log](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/MySQL-InnoDB事务的实现/Insert-undo-log.png)

3.3.2 Update undo log

Update undo log记录的是对delete和update操作产生的undo log。该undo log可能需要提供MVCC机制，因此不能在事务提交时就进行删除。提交时放入undo log链表，等待purge线程进行最后的删除。

Update undo log相对于Insert Undo log记录的内容更多，所需占用的空间更大。其结构如下图所示：

![Update-undo-log](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/MySQL-InnoDB事务的实现/Update-undo-log.png)

Type_cmpl，其可能的值包括：

- ①、12 TRX_UNDO_UPD_EXIST_REC更新non-delete-mark的记录
- ②、13 TRX_UNDO_UPD_DEL_REC将delete的记录标记为not delete
- ③、14 TRX_UNDO_DEL_MARK_REC将记录标记为delete

Update_vector信息表示update操作发生改变的列。每个修改的列信息都要记录到undo log中，对于不同的undo log类型，可能还需要记录对索引列所做的修改。

3.4  查看Undo 信息

Oracle和Microsoft SQL Server数据库都是由内部的数据字典来观察当前undo的信息。InnoSQL对information_schema进行了扩展，添加了两张数据字典表，方便查看undo信息。涉及的字典表为：

- ①、INNODB_TRX_ROLLBACK_SEGMENT：查看rollback segment
- ②、INNODB_TRX_UNDO：查看记录事务对应undo log

INNODB_TRX_ROLLBACK_SEGMENT 其表结构如下所示（执行书上的命令时发现字典不存在...？？？？？？）

mysql> desc INNODB_TRX_ROLLBACK_SEGMENT; ERROR 1146 (42S02): Table 'sys.INNODB_TRX_ROLLBACK_SEGMENT' doesn't exist

按照书中演示的使用INNODB_TRX_UNDO表

①、创建测试表

mysql> create table t(a int,b varchar(32), primary key(a), key(b))engine=innodb; Query OK, 0 rows affected (0.09 sec)

②、插入一条记录，并尝试通过INNODB_TRX_UNDO观察该事务的undo log的情况

mysql> start transaction; Query OK, 0 rows affected (0.00 sec) mysql> insert into t select 1, '1'; Query OK, 1 row affected (0.01 sec) Records: 1  Duplicates: 0  Warnings: 0 mysql> select * from information_schema.INNODB_TRX_UNDO\G; ERROR 1109 (42S02): Unknown table 'INNODB_TRX_UNDO' in information_schema

阿西吧，此时发现该字典表也不存在....?????????

Delete操作并不直接删除记录，而只是将记录标记为已删除，也就是将记录的delete flag设置为1。而记录最终的删除是在purge操作中完成的。

Update操作实际分为两步：

- ①、首先将原主键记录标记为已删除，因此需产生一个类型为TRX_UNDO_DEL_MARK_REC的undo log。
- ②、然后插入一条新的记录，因此需要产生一个类型为TRX_UNDO_INSERT_REC的undo log。

4 Purge

Purge用于最终完成delete和update操作，这样设计是因为InnoDB存储引擎支持MVCC，所以记录不能在事务提交时立即处理。其操作示例如下图所示：

![undo-log与history列表的关系](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/MySQL-InnoDB事务的实现/undo-log与history列表的关系.png)

Innodb提供了一个全局动态参数用于控制purge操作：

- ①、innodb_purge_batch_size：用于设置每次purge操作需要清理的undo page数量，默认值为300。推荐采用默认值。
- ②、innodb_max_purge_lag：用于控制history list的长度，若长度大于该参数，其会延缓”DML“的操作。该参数默认值为0，表示不对history list做任何限制。当大于0时，就会延缓DML的操作，其延缓单位为毫秒，延缓算法为：

delay = ((length(history_list) - innodb_max_purge_lag) * 10) - 5

注：delay的对象是行，而不是一个DML操作，即假设update操作5行数据时，总延时时间为5*delay。

- ③、innodb_max_purge_lag_delay：要求版本大于InooDB1.2版本，用于控制delay的最大毫秒数。即当计算的delay值大于此参数时，将delay设置为此参数，避免由于purge操作缓慢导致其他SQL线程出现无限制的等待。

5 Group Commit

Group commit操作用于提高磁盘fsync的效率，即一次fsync操作可以刷新确保多个事务日志被写入文件。

对于InnoDB存储引擎，事务提交时会进行两个阶段的操作：

- ①、修改内存中事务对应的信息，并将日志写入重做日志缓冲。
- ②、调用fsync将去报日志都从重做日志缓冲写入磁盘。

其中第②步较慢，因此存储引擎需要和磁盘打交道。对于在执行第②段操作时，其他的事务可以进行第一段操作，然后将多个事务的重做日志通过一次fsync刷新到磁盘，大大减少磁盘的压力，提高数据库性能。对于Insert和update较为明显的操作，Group Commit的效果尤为明显。

Q3：在InnoDB1.2版本以后，在开启二进制日志后，InnoDB的Group Commit功能会实现，导致性能下降，其原因为？

导致这个问题的原因在于开启二进制日志后，为了保证存储引擎层中的事务和二进制日志的一致性，二者之前使用两阶段事务，其步骤如下：

- ①、当事务提交时，InnoDB存储引擎进行prepare操作。

- ②、Mysql数据库上层写二进制日志。（一旦这一步提交，就确保了事务的提交，即使在第③步中发生了宕机。）（这一步由参数sync_binlog控制）

- ③、InnoDB存储引擎层将日志写入重做日志文件。（这一步的fsync由参数innodb_flush_log_at_trx_commit控制）

- - a、修改内存中事务对应的信息，并将日志写入重做日志缓冲。
  - b、调用fsync将确保日志都从重做日志缓冲写入磁盘。

注：每个步骤都需要fsync操作才能保证上下两层数据的一致性。

其整体过程如下：

![开启二进制日志后InnoDB存储引擎的提交过程](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/MySQL-InnoDB事务的实现/开启二进制日志后InnoDB存储引擎的提交过程.png)

为了保证二进制日志的写入顺序与InnoDB层的事务提交顺序一致，Mysql数据库内部使用了prepare_commit_mutex锁，当开启这个锁后，步骤③中的a将不可以在其他事务中执行步骤b，从而导致了Group Commit失效。

Q4：如何解决Q3的问题？

Mysql5.6采用了Binary Log Group Commit（BLGC）的方式将事务提交的过程采用三段式提交方式，如下图所示：

![BLGC](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/MySQL-InnoDB事务的实现/BLGC.png)

当有一组事务在今次那个Commit阶段时，其他新事物可以进行Flush阶段，从而使Group Commit不断生效。

InnoDB提供了一些参数用于控制Bin log提交：

①、binlog_max_flush_quenue_time：用于控制Flush阶段中等待的时间，即使之前一组事务完成提交，当前一组是物业部马上进入Sync阶段，而是至少等待一端时间，这样做的好处是Group Commit的事务数量更过，缺点是可能会导致事务的响应时间变量。默认值为0，且不建议修改。

**参考资料**

- \1. MySQL技术内幕+InnoDB存储引擎第2版(7.2节)
- 2.[【MySQL】SHOW ENGINE INNODB STATUS \G之Pages flushed up to的理解](http://blog.itpub.net/30221425/viewspace-2154670/)