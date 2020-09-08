---
title: MySQL-为什么使用B+树作为索引结构
date: 2020-09-07 17:03:33
tags: ["MySQL","索引"]
categories: ["MySQL", "索引"]
---

1 概述

首先,MySQL和B+树是没有直接关系的，真正与B+树有关系的是MySQL的默认存储引擎，MySQL中存储引擎的主要作用是负责数据的存储和提取．除了InnoDB外，MySQL还支持另外一些存储引擎，使用`show engines`命令可查看．

```mysql
mysql> show engines;
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| Engine             | Support | Comment                                                        | Transactions | XA   | Savepoints |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| MyISAM             | YES     | MyISAM storage engine                                          | NO           | NO   | NO         |
| MRG_MYISAM         | YES     | Collection of identical MyISAM tables                          | NO           | NO   | NO         |
| InnoDB             | DEFAULT | Supports transactions, row-level locking, and foreign keys     | YES          | YES  | YES        |
| BLACKHOLE          | YES     | /dev/null storage engine (anything you write to it disappears) | NO           | NO   | NO         |
| PERFORMANCE_SCHEMA | YES     | Performance Schema                                             | NO           | NO   | NO         |
| CSV                | YES     | CSV storage engine                                             | NO           | NO   | NO         |
| ARCHIVE            | YES     | Archive storage engine                                         | NO           | NO   | NO         |
| MEMORY             | YES     | Hash based, stored in memory, useful for temporary tables      | NO           | NO   | NO         |
| FEDERATED          | NO      | Federated MySQL storage engine                                 | NULL         | NULL | NULL       |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
9 rows in set (0.24 sec)
```

对于InnoDB来说，所有数据都是以键值对的方式存储在Ｂ+树结构中，其中表中的数据（主键索引）以<id, row>的方式存储；而辅助索引以<index, id>的方式进行存储．

- 在主键索引中，ｉｄ是主键，通过ｉｄ能找到该行的全部列．
- 在辅助索引中，索引中的几个列构成了键，通过索引中的列找到ｉｄ，如果有需要的话，可以载通过ｉｄ找到当前数据行的全部内容．



2 分析

MySQL作为OLTP(Online Tranaction Processing)的数据库，不仅需要具备事务的处理能力，还要保证数据的持久化并且能够有一定的实时数据查询能力．

一般来说索引文件本身也很大，不可能全部存储在内存中，因此索引往往以索引文件的形式存储在磁盘上．这样的化，索引查找过程就会产生磁盘I/O消耗，相对于内存的存取，I/Ｏ存取的消耗要高几个数量级，所以评判一个数据结构作为索引的优劣最重要的指标就是在查找过程中磁盘I/O操作次数的渐进复杂读．换句话说，索引的结构组织要尽量减少查找过程中磁盘I/O的存取次数．



对于普通磁盘（非ＳＳＤ）加载数据需要经过队列，寻道，旋转以及传输这些过程，大约耗时为１０ｍｓ左右，如下图所示：

![磁盘IO消耗](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/MySQL-为什么使用B+树作为索引结构/磁盘IO消耗.png)



对于InnoDB选择B+树索引结构原因可以从以下几个方面进行分析：

- 考虑其支持的场景和功能需要在特定查询上拥有较强的性能；
- CPU将磁盘上的数据加载到内存需要花费大量的时间，采用Ｂ+树将是非常好的选择；



2.1 读写性能分析

这里选取哈希结构及Ｂ树结构进行对比分析．

2.1.1 与哈希结构进行比较

若使用Ｂ+树作为底层的数据结构，其增删改查一条数据的ＳＱＬ的时间复杂度都是O(lg n)，即树的高度．

若使用哈希作为底层的数据结构，其增删改查一条数据的ＳＱＬ的时间复杂度可能达到Ｏ(1)．

对于单行数据的读写，哈希索引均比Tree型索引块，但是对于某些特殊的查询SQL，例如分组、排序、比较等情况，哈希索引的时间复杂度会退化为O(n)，而Tree索引因为其有序的特性，能依旧保持O(lg(n))的高效率。



2.2 与Ｂ树进行比较

Ｂ树和Ｂ+树在数据结构上起始有一些相似的，都能按照某些顺序对索引中内容进行遍历，排序及范围查找，相比哈希结构来说，Ｂ树和Ｂ+树能带来更好的性能．



2.2 数据加载

哈希结构无法满足一些特殊的查询SQL，而Ｂ树和Ｂ+树均能相对高效的执行，那么为什么不选择Ｂ树呢？

Ｂ树与Ｂ+树最大的区别就是，Ｂ树的非页结点中可以存储数据，但是Ｂ+树所有数据都是存储于叶子结点中．当一个表的底层数据结构是Ｂ树时，假设我们需要访问索引在［4, 9］区间的数据：

![b-tree](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/MySQL-为什么使用B+树作为索引结构/b-tree.png)

在不考虑优化的情况下，在上面简单Ｂ树中需要进行４次磁盘的随机I/O访问才能找到所有满足条件的行：

- 加载根结点所在的页，发现根结点的第一个元素是６，大于４；
- 通过根节点的指针加载左字节点所在的页，遍历页面中的数据，找到５；
- 重新加载根节点所在的页，发现根节点不包含第二个元素；
- 通过根节点的指针加载右子节点所在的页，遍历页面中的数据，找到７，８；

（*以上过程不考虑对过程进行优化，Ｂ树能做的优化Ｂ＋树都能做*）

















------

参考资料

1 [为什么MySQL使用B+树](https://draveness.me/whys-the-design-mysql-b-plus-tree/)