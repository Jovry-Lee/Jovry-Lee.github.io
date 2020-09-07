---
title: MySQL-为什么使用B+树作为索引结构
date: 2020-09-07 17:03:33
tags: ["MySQL","索引"]
categories: ["MySQL", "索引"]
---













1 概述

首先,MySQL和B+树是没有直接关系的，真正与B+树有关系的是MySQL的默认存储引擎，MySQL中存储引擎的主要作用是负责数据的存储和提取．除了InnoDB外，MySQL还支持另外一些存储引擎，使用show engines命令可查看．

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

对于InnoDB来说，所有数据都是以键值对的方式存储在Ｂ+树结构中，其中表中的数据（主键索引）以＜ｉｄ，ｒｏｗ＞的方式存储；而辅助索引以＜ｉｎｄｅｘ，ｉｄ＞的方式进行存储．

- 在主键索引中，ｉｄ是主键，通过ｉｄ能找到该行的全部列．
- 在辅助索引中，索引中的几个列构成了键，通过索引中的列找到ｉｄ，如果有需要的话，可以载通过ｉｄ找到当前数据行的全部内容．





















------

参考资料

1 [为什么MySQL使用B+树](https://draveness.me/whys-the-design-mysql-b-plus-tree/)