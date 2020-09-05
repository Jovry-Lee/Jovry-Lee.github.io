---
title: MySQL-Explain详解
date: 2020-09-05 11:17:41
tags: ["MySQL"]
categories: ["MySQL"]
---



### 1 简介

`EXPLAIN`命令是`查看查询优化器如何决定执行查询的主要方法`。虽然这个功能有一些局限性，但是它的输出是可以获取的最好信息。在使用EXPLAIN时，MySQL会在查询上设置一个标记，当执行查询时，这个标记会使其返回关于执行计划中每一步的信息，而不是执行它。它会返回一行或多行信息，显示出执行计划中的每一部分和执行次序。

<!--more-->

注意：EXPLAIN只是个近似结果，有时候它是一个很好的近似，但有时候也会相差甚远。



以下是使用EXPLIAN的一些相关限制：

- 不会告诉你触发器、存储过程或UDF会如何影响查询；
- 不支持存储过程，尽管可以手动抽取查询并单独的对其进行EXPLAIN操作；
- 不会告诉你MySQL在查询执行中所作的特定优化；
- 不会显示关于查询的执行计划的所有信息；
- 不区分具有相同名字的事物。例如：对于内存排序和临时文件都使用“filesort”，并且对于磁盘上和内存中的临时表都显示“Using temporary”；
- 可能会出现误导。例如，`对于一个有着很小的limit查询显示全索引扫描`。



### 1.1 重写非SELECT查询

EXPLAIN只能解释SELECT查询，并不会对存储程序调用、INSERT、UPDATE、DELETE或其他语句做解释。

解决方法：重写非SELECT查询以利用EXPLAIN，



### 2 用法

```
EXPLAIN tbl_name # 得出一个表的字段结构。
EXPLAIN [EXTENDED] SELECT select_options # 给出当前SQL语句相关的一些信息。 
```

以`virtual_coins表`为例，后续测试均在此表基础上进行验证。

```mysql
CREATE TABLE `virtual_coins` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT COMMENT '自增主键',
  `uid` int(10) NOT NULL DEFAULT '0' COMMENT '用户ID',
  `type` varchar(20) NOT NULL DEFAULT '' COMMENT '虚拟货币类型',
  `channel` varchar(20) NOT NULL DEFAULT '' COMMENT '渠道',
  `total_coins` int(10) NOT NULL DEFAULT '0' COMMENT '当前虚拟货币类型总数',
  `usable_coins` int(10) NOT NULL DEFAULT '0' COMMENT '当前虚拟货币类型可用数量',
  `frozen_coins` int(10) NOT NULL DEFAULT '0' COMMENT '当前虚拟货币类型冻结数量',
  `status` tinyint(4) NOT NULL DEFAULT '1' COMMENT '状态：1-有效，0-无效（封禁）',
  `create_time` int(10) NOT NULL DEFAULT '0' COMMENT '创建时间',
  `update_time` int(10) NOT NULL DEFAULT '0' COMMENT '更新时间',
  PRIMARY KEY (`id`),
  KEY `idx_uid_type` (`uid`,`type`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='虚拟货币快照表'
```

```mysql
mysql> explain virtual_coins;
+--------------+------------------+------+-----+---------+----------------+
| Field        | Type             | Null | Key | Default | Extra          |
+--------------+------------------+------+-----+---------+----------------+
| id           | int(11) unsigned | NO   | PRI | NULL    | auto_increment |
| uid          | int(10)          | NO   | MUL | 0       |                |
| type         | varchar(20)      | NO   |     |         |                |
| channel      | varchar(20)      | NO   |     |         |                |
| total_coins  | int(10)          | NO   |     | 0       |                |
| usable_coins | int(10)          | NO   |     | 0       |                |
| frozen_coins | int(10)          | NO   |     | 0       |                |
| status       | tinyint(4)       | NO   |     | 1       |                |
| create_time  | int(10)          | NO   |     | 0       |                |
| update_time  | int(10)          | NO   |     | 0       |                |
+--------------+------------------+------+-----+---------+----------------+
10 rows in set (0.00 sec)
```


```mysql
mysql> explain select * from virtual_coins where uid=2000034000;
+----+-------------+---------------+------------+------+---------------+--------------+---------+-------+------+----------+-------+
| id | select_type | table         | partitions | type | possible_keys | key          | key_len | ref   | rows | filtered | Extra |
+----+-------------+---------------+------------+------+---------------+--------------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | virtual_coins | NULL       | ref  | idx_uid_type  | idx_uid_type | 4       | const |    1 |   100.00 | NULL  |
+----+-------------+---------------+------------+------+---------------+--------------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```



### 3 字段值详解

#### 3.1 ID列

id列用于标识select所属的行。若在语句中没有子查询或联合查询，那么只会有唯一的select，于是每一行在这个列中将显示一个1。否则，内层的select语句一般会顺序编号，对应于其在原始语句中位置。

示例：

```
mysql> explain select uid from (select uid from tuanmei_user) as vc;
```

| id   | select_type | table         | type  | possible_keys | key          | key_len | ref  | rows  |    Extra    |
| ---- | ----------- | ------------- | ----- | ------------- | ------------ | ------- | ---- | ----- | :---------: |
| 1    | PRIMARY     |               | ALL   | NULL          | NULL         | NULL    | NULL | 41171 |    NULL     |
| 2    | DERIVED     | virtual_coins | index | NULL          | idx_uid_type | 66      | NULL | 41171 | Using index |

#### 3.2 SELECT_TYPE列

查询的类型，主要是`区别简单查询和联合查询、子查询之类的复杂查询`。  

常见的取值：

  - `SIMPLE`
    它表示简单的 select，不包括 union 和子查询；
    
  - `PRIMARY`
    查询有任何复杂的子查询，则最外层部分标记为 primary；
    
  - `SUBQUERY`
    
    包含在select列表中的子查询中的select（即，不在from子句中）标记为`subquery`。
    
    ```mysql
    mysql> explain select (select uid from virtual_coins where uid = 2000163293);
    +----+-------------+---------------+------------+------+---------------+--------------+---------+-------+------+----------+----------------+
    | id | select_type | table         | partitions | type | possible_keys | key          | key_len | ref   | rows | filtered | Extra          |
    +----+-------------+---------------+------------+------+---------------+--------------+---------+-------+------+----------+----------------+
    |  1 | PRIMARY     | NULL          | NULL       | NULL | NULL          | NULL         | NULL    | NULL  | NULL |     NULL | No tables used |
    |  2 | SUBQUERY    | virtual_coins | NULL       | ref  | idx_uid_type  | idx_uid_type | 4       | const |    1 |   100.00 | Using index    |
    +----+-------------+---------------+------------+------+---------------+--------------+---------+-------+------+----------+----------------+
    2 rows in set, 1 warning (0.14 sec)
    ```
    
  - `DERIVED`

      表示包含在from子句的子查询中的select，MySQL会递归执行并将结果放到一个临时表中。（测试时，只有在数据量大的情况下，才会出现DERIVED，数据量小的时候是SIMPLE??????）

      ```
      mysql> explain select uid from (select uid from tuanmei_user) as vc;
      ```

      | id   | select_type | table         | type  | possible_keys | key          | key_len | ref  | rows  |    Extra    |
      | ---- | ----------- | ------------- | ----- | ------------- | ------------ | ------- | ---- | ----- | :---------: |
      | 1    | PRIMARY     |               | ALL   | NULL          | NULL         | NULL    | NULL | 41171 |    NULL     |
      | 2    | DERIVED     | virtual_coins | index | NULL          | idx_uid_type | 66      | NULL | 41171 | Using index |

  - `UNION`
    union 语句的第二个或者说是后面那一个select被标记为union。
    
    ```mysql
    mysql> explain select * from virtual_coins where id <100 union select * from virtual_coins where id >= 100;
    +----+--------------+---------------+------------+-------+---------------+---------+---------+------+------+----------+-----------------+
    | id | select_type  | table         | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra           |
    +----+--------------+---------------+------------+-------+---------------+---------+---------+------+------+----------+-----------------+
    |  1 | PRIMARY      | virtual_coins | NULL       | range | PRIMARY       | PRIMARY | 4       | NULL |   55 |   100.00 | Using where     |
    |  2 | UNION        | virtual_coins | NULL       | range | PRIMARY       | PRIMARY | 4       | NULL |   52 |   100.00 | Using where     |
    | NULL | UNION RESULT | <union1,2>    | NULL       | ALL   | NULL          | NULL    | NULL    | NULL | NULL |     NULL | Using temporary |
    +----+--------------+---------------+------------+-------+---------------+---------+---------+------+------+----------+-----------------+
    3 rows in set, 1 warning (0.44 sec)
    ```
    
  - `DEPENDENT`

      表示select依赖于外层查询中的数据。

  - `UNION RESULT`

      用来从UNION的匿名临时表检索结果的select被标记为UNION RESULT。

#### 3.3 TABLE 

所使用的表。

#### 3.4 TYPE

关联类型（访问类型），MySQL决定如何查找表中的行，<u>是较为重要的一个指标</u>。

  - `NULL`
    
意味着MySQL能在优化阶段分解查询语句，在执行阶段甚至用不着再访问表或者索引。
    
    ```mysql
    mysql> explain select 1;
    +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
    | id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra          |
    +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
    |  1 | SIMPLE      | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | No tables used |
    +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
    1 row in set, 1 warning (0.11 sec)
    ```
    
  - `system`
    表仅有一行，是 const 类型的特例。不常见。

  - `const`  
    常量查询，在整个查询过程中这个表`最多只会有一条匹配的行`，`用到了 primary key 或者unique 索引`。  
  
  
  
  eg：
```mysql
  mysql> explain select * from virtual_coins where id=50;
  +----+-------------+---------------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
  | id | select_type | table         | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
  +----+-------------+---------------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
  |  1 | SIMPLE      | virtual_coins | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL  |
  +----+-------------+---------------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
  1 row in set, 1 warning (0.00 sec)
```


  - `eq_ref` 
    对于每个来自于前面的表的行组合，从该表中读取一行。 （*个人理解是在联合查询时，使用的条件是 UNIQUE 或 PRIMARY KEY。*）
```mysql
  mysql> explain select total_coins from virtual_coins, virtual_coins_logs where virtual_coins.id = virtual_coins_logs.id;
  +----+-------------+--------------------+------------+--------+---------------+---------+---------+--------------------------------+------+----------+--------------------------+
  | id | select_type | table              | partitions | type   | possible_keys | key     | key_len | ref                            | rows | filtered | Extra                    |
  +----+-------------+--------------------+------------+--------+---------------+---------+---------+--------------------------------+------+----------+--------------------------+
  |  1 | SIMPLE      | virtual_coins      | NULL       | ALL    | PRIMARY       | NULL    | NULL    | NULL                           |   41 |   100.00 | NULL                     |
  |  1 | SIMPLE      | virtual_coins_logs | NULL       | eq_ref | PRIMARY       | PRIMARY | 8       | fans_economic.virtual_coins.id |    1 |   100.00 | Using where; Using index |
  +----+-------------+--------------------+------------+--------+---------------+---------+---------+--------------------------------+------+----------+--------------------------+
  2 rows in set, 1 warning (0.00 sec)
```
  - `ref` 
    
  这是一种索引访问，也叫索引查找，它返回所有匹配某个单个值的行。对于每个来自于前面的表的行组合，所有有匹配索引值的行将从这张表中读取。
  
  如果联接只使用键的最左边的前缀，或如果键不是 UNIQUE 或 PRIMARY KEY（换句话说，如果联接不能基于关键字选择单个行的话），则使用 ref。如果使用的键仅仅匹配少量行，该联接类型是不错的。
  
  - `fulltext`  

  - `ref_or_null`
    该联接类型如同 ref，但是添加了 MySQL 可以专门搜索包含 NULL 值的行。 在解决子查询中经常使用该联接类型的优化。

  - `index_merge`  
  - `unique_subquery`  
  - `index_subquery`  
  - `range`
    给定范围内的检索，使用一个索引来检查行。通常发生在在索引列上使用范围查询，如 >，<，in 等时，非索引列是 ALL。
```mysql
  mysql> explain select * from virtual_coins where id in (10, 100);
  +----+-------------+---------------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
  | id | select_type | table         | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
  +----+-------------+---------------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
  |  1 | SIMPLE      | virtual_coins | NULL       | range | PRIMARY       | PRIMARY | 4       | NULL |    2 |   100.00 | Using where |
  +----+-------------+---------------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
  1 row in set, 1 warning (0.00 sec)

  mysql> explain select * from virtual_coins where uid in (10, 100);
  +----+-------------+---------------+------------+-------+---------------+--------------+---------+------+------+----------+-----------------------+
  | id | select_type | table         | partitions | type  | possible_keys | key          | key_len | ref  | rows | filtered | Extra                 |
  +----+-------------+---------------+------------+-------+---------------+--------------+---------+------+------+----------+-----------------------+
  |  1 | SIMPLE      | virtual_coins | NULL       | range | idx_uid_type  | idx_uid_type | 4       | NULL |    2 |   100.00 | Using index condition |
  +----+-------------+---------------+------------+-------+---------------+--------------+---------+------+------+----------+-----------------------+
  1 row in set, 1 warning (0.00 sec)
  
  mysql> explain select * from virtual_coins_0 where total_coins in (0,10);
  +----+-------------+-----------------+------------+------+---------------+------+---------+------+------+----------+-------------+
  | id | select_type | table           | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
  +----+-------------+-----------------+------------+------+---------------+------+---------+------+------+----------+-------------+
  |  1 | SIMPLE      | virtual_coins_0 | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    1 |   100.00 | Using where |
  +----+-------------+-----------------+------------+------+---------------+------+---------+------+------+----------+-------------+
  1 row in set, 1 warning (0.00 sec)
```

  - `index` 
    按索引次序扫描，先读索引，再读实际的行，结果也是全表扫描，主要优点是避免了排序。（索引是排好序的，并且 all 是从硬盘中读的，index 可能不在硬盘上）
```mysql
  mysql> explain select * from virtual_coins order by id;
  +----+-------------+---------------+------------+-------+---------------+---------+---------+------+------+----------+-------+
  | id | select_type | table         | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra |
  +----+-------------+---------------+------------+-------+---------------+---------+---------+------+------+----------+-------+
  |  1 | SIMPLE      | virtual_coins | NULL       | index | NULL          | PRIMARY | 4       | NULL |   41 |   100.00 | NULL  |
  +----+-------------+---------------+------------+-------+---------------+---------+---------+------+------+----------+-------+
  1 row in set, 1 warning (0.00 sec)
```

  - `ALL` 
    进行完整的表扫描。性能很差，通常可以增加更多的索引而不要使用 ALL，使得行能基于前面的表中的常数值或列值被检索出。
```mysql
  mysql> explain select * from virtual_coins where create_time=111;
  +----+-------------+---------------+------------+------+---------------+------+---------+------+------+----------+-------------+
  | id | select_type | table         | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
  +----+-------------+---------------+------------+------+---------------+------+---------+------+------+----------+-------------+
  |  1 | SIMPLE      | virtual_coins | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   41 |    10.00 | Using where |
  +----+-------------+---------------+------------+------+---------------+------+---------+------+------+----------+-------------+
  1 row in set, 1 warning (0.00 sec)
```

**结果值优先顺序**为：  
    `NULL > system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL`



一般来说，得保证查询至少达到 range 级别，最好能达到 ref。

#### 3.5 POSSIBLE_KEY列

提示使用哪个索引会在该表中找到行，不太重要。

#### 3.6 KEY列

显示了MySQL决定采用哪个索引来优化对该表的访问。若该索引不在possible_keys中，那么MySQL选用它是处于另外的原因，例如，它可能选择了一个覆盖索引，哪怕没有where条件。

如果是 NULL，表示没有索引被选择。

#### 3.7 KEY_LEN

使用的索引字节数。

#### 3.8 REF

显示了之前的表再key列记录的索引中查找值所用的列或常量。

#### 3.9 ROWS 

显示执行查询的行数（而不是返回的行数），数值越大越不好，说明没有用好索引。但对 InnoDB 不太准。

#### 3.10 `EXTRA`  

该列包含的是其他的额外信息。其中常见的重要的值如下：

- `Using index` 
此查询使用了`覆盖索引（Covering Index）`，即通过索引就能返回结果，无需访问表。
```mysql
mysql> explain select id from virtual_coins;
+----+-------------+---------------+------------+-------+---------------+--------------+---------+------+------+----------+-------------+
| id | select_type | table         | partitions | type  | possible_keys | key          | key_len | ref  | rows | filtered | Extra       |
+----+-------------+---------------+------------+-------+---------------+--------------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | virtual_coins | NULL       | index | NULL          | idx_uid_type | 66      | NULL |   41 |   100.00 | Using index |
+----+-------------+---------------+------------+-------+---------------+--------------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

- `Using where`
表示 MySQL 服务器从存储引擎收到行后再进行`“后过滤”（Post-filter）`。所谓“后过滤”，就是先读取整行数据，再检查此行是否符合 where 句的条件，符合就留下，不符合便丢弃。因为检查是在读取行后才进行的，所以称为“后过滤”。查询中含有 WHERE 子句时较常见。
```mysql
mysql> explain select * from virtual_coins where create_time=111;
+----+-------------+---------------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table         | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+---------------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | virtual_coins | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   41 |    10.00 | Using where |
+----+-------------+---------------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

- `Using filesort` 

  表示MySQL会对结果使用一个外部索引排序，而不是按索引次序从表里读取行。MySQL有两种文件排序的算法，两种方法都可以在内存或者磁盘上完成。Using filesort并不会对其进行区分。

  

  例如下面，id 是主键，所以它是索引。当 order by id 时，Extra 是 Using Index，而对别的列进行排序，就是 Using filesort，表示在查询之后，又进行了一次排序。

```mysql
#  Using index（对索引列排序）:
mysql> explain select id from virtual_coins order by id;
+----+-------------+---------------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type | table         | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+---------------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | virtual_coins | NULL       | index | NULL          | PRIMARY | 4       | NULL |   41 |   100.00 | Using index |
+----+-------------+---------------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

# Using filesort（对非索引列排序）:
mysql> explain select * from virtual_coins_0 order by total_coins;
+----+-------------+-----------------+------------+------+---------------+------+---------+------+------+----------+----------------+
| id | select_type | table           | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra          |
+----+-------------+-----------------+------------+------+---------------+------+---------+------+------+----------+----------------+
|  1 | SIMPLE      | virtual_coins_0 | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    1 |   100.00 | Using filesort |
+----+-------------+-----------------+------------+------+---------------+------+---------+------+------+----------+----------------+
1 row in set, 1 warning (0.00 sec)
```

- `Using temporary` 
`需要创建临时表存储结果以完成查询`。这种情况通常发生在查询时包含了 `Group By` 和 `Order By` 子句时或者`联合查询`时。

```mysql
mysql> explain select * from virtual_coins_logs group by uid order by id;
+----+-------------+--------------------+------------+-------+---------------------+---------------------+---------+------+------+----------+---------------------------------+
| id | select_type | table              | partitions | type  | possible_keys       | key                 | key_len | ref  | rows | filtered | Extra                           |
+----+-------------+--------------------+------------+-------+---------------------+---------------------+---------+------+------+----------+---------------------------------+
|  1 | SIMPLE      | virtual_coins_logs | NULL       | index | idx_uid_type_action | idx_uid_type_action | 218     | NULL | 2701 |   100.00 | Using temporary; Using filesort |
+----+-------------+--------------------+------------+-------+---------------------+---------------------+---------+------+------+----------+---------------------------------+
1 row in set, 1 warning (0.00 sec)
```

- `range checked for each record (index map:N)`

  表示没有好用的索引，新的索引将在联接的每一行上重新估算。N是显示在possible_keys列中索引的位图，并且是冗余的。

---
### 总结

- explain 的四个重要字段：`type、key、rows、extra`；
- 如果 type 的值为 index 或者 ALL，那么说明该 SQL 性能一般，需要优化；

- 如果 key 的值为 NULL，说明该 SQL 没有使用索引，可以考虑在关键字段上增加索引；

- row 的值代表了进行本次查询时，搜索记录的条数，当这个值特别大的时候，说明该 SQL 语句性能差；

- 如果 Extra 字段的值为 Using filesort 或 Using temporary，也是需要优化的，可以通过调整 order by 或者 group by 的字段来实现；

- 联合查询时，一定要多用 explain 来查看查询性能



------

### 参考资料：

高性能MySQL 第三版