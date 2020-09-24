---
title: MySQL-索引分类
date: 2020-09-06 01:04:57
tags: ["MySQL","索引"]
categories: ["MySQL"]
---

### 1 从逻辑角度划分类

索引本质上是表字段的有序子集，其每个记录项指向相应的表记录。MySQL共有4类索引：

<!--more-->

- 主键索引
- 唯一索引
- 常规索引
- 全文索引



#### 1.1 主键索引

`主键索引用于根据主键自身的唯一性来唯一标识每条记录。`因此，该键必须是该记录所表示实体唯一拥有的值，或者是数据库生成的唯一值，例如，自增整数。



示例: 设置id作为自增主键索引.

```mysql
create table `bookmarks` (
    `id` int unsigned not null auto_increment comment '主键id',
    `name` varchar(75) not null default '' comment '名称',
    `url` varchar(200) not null default '' comment 'URL',
    `description` mediumtext not null comment '描述',
    primary key(`id`)
) engine=InnoDB default charset=utf8mb4 comment='书签表';
```

注：<u>每个表只能有一个自增字段,且该字段必须为主键,为主键的字段不能包含null值,即使没有显式申明为not null, MySQL也会自动赋予此特性.</u>



#### 1.2 唯一索引

与主键索引一样，唯一索引可以预防哪个创建重复的值。但是，<u>与之不同之处在于每个表只能有一个主键索引,但可以有多个唯一索引.可以为null值.</u>



示例:设置name和url作为唯一联合索引

```mysql
create table `bookmarks_v1` (
    `id` int unsigned not null auto_increment comment '主键id',
    `name` varchar(75) not null default '' comment '名称',
    `url` varchar(200) not null default '' comment 'URL',
    `description` mediumtext not null comment '描述',
    primary key (`id`),
    unique key `name_url` (`name`,`url`)
) engine=InnoDB default charset=utf8mb4 comment='书签表';
```



#### 1.3 普通索引

普通索引可以分为`单列普通索引`和`联合普通索引`.

##### 1.3.1 单列普通索引

若表中的某个列将成为大量选择查询的焦点，就应当使用单列普通索引。

示例：设置email为唯一索引，姓氏为单例普通索引。

```mysql
create table employees(
    `id` int unsigned not null auto_increment comment '主键id',
    `firstname` varchar(100) not null default '' comment '名称',
    `lastname` varchar(100) not null default '' comment '姓氏',
    `email` varchar(100) not null default '' comment '电子邮箱',
    primary key (`id`),
    unique key `idx_email` (`email`),
    key `idx_lastname` (`lastname`)
) engine=InnoDB default charset=utf8mb4 comment='员工信息表';
```

##### 1.3.2 多列普通索引(联合索引)

若在查询中，经常会使用多列一起使用，则可使用联合索引，MySQL的联合索引方法基于最左前缀的策略.



假设为列A、B、C添加联合索引，则使用下列组合可以提高涉及的查询性能：

```
A,B,C
A,B
A
```



示例:

```mysql
create table employees_v1(
    `id` int unsigned not null auto_increment comment '主键id',
    `firstname` varchar(100) not null default '' comment '名称',
    `lastname` varchar(100) not null default '' comment '姓氏',
    `email` varchar(100) not null default '' comment '电子邮箱',
    primary key (`id`),
    unique key `idx_email` (`email`),
    key `idx_lastname_firstname` (`lastname`, `firstname`)
) engine=InnoDB default charset=utf8mb4 comment='员工信息表';
```



#### 1.4 全文索引

全文索引提供了一种高效的方法来搜索存储为`char`，`varchar`或者`text`数据类型的文本。



示例：在`description`字段上添加全文索引。

```mysql
create table `bookmarks_v2` (
    `id` int unsigned not null auto_increment comment '主键id',
    `name` varchar(75) not null default '' comment '名称',
    `url` varchar(200) not null default '' comment 'URL',
    `description` mediumtext not null comment '描述',
    primary key(`id`),
    fulltext key `idx_description`(`description`)
) engine=InnoDB default charset=utf8mb4 comment='书签表';
```

基于全文索引获取数据时，select查询使用两个特殊的MySQL函数`MATCH()`和`AGAINST()`。利用这两个函数，可以针对全文索引执行自然语言搜索，如下所示：

- 添加数据


```mysql
insert into bookmarks_v2 (name,url,description) 
    values 
    ("Python", "www.python.org", "The official Python Web site"),
    ("MySQL manuel", "http://dev.mysql.com/doc", "The MySQL reference manual"),
    ("Apache site", "http://httpd.apache.org", "Includes Apache 2 manual"),
    ("PHP Hypertext", "www.php.net", "The official PHP Web site"),
    ("Apache Week", "www.apacheweek.com", "Offers a dedicated Apache 2 section");
```



- 根据全文索引查询

```mysql
mysql> select name,url FROM bookmarks_v2 where match(description) against('Apache 2');
+-------------+-------------------------+
| name        | url                     |
+-------------+-------------------------+
| Apache site | http://httpd.apache.org |
| Apache Week | www.apacheweek.com      |
+-------------+-------------------------+
2 rows in set (0.07 sec)

mysql> select match(description) against('Apache 2') from bookmarks_v2;
+----------------------------------------+
| match(description) against('Apache 2') |
+----------------------------------------+
|                                      0 |
|                                      0 |
|                    0.15835624933242798 |
|                                      0 |
|                    0.15835624933242798 |
+----------------------------------------+
5 rows in set (0.05 sec)
```

该查询列出在description中出现了"Apache"的记录，以相关性从高到底的顺序排序。(其中"2"由于长度过短,因此被忽略了).



当`MATCH()`用于`WHERE`子句时，相关性按照返回的记录与搜索的字符串的匹配程度来定义。

<u>MATCH和AGAINST函数也可以放在查询体中,返回匹配记录的加权列表,分数越高,相关性就越大</u>.



### 2 索引的创建方式

创建索引的方式有两种: 

- **alter table**
- **create table**



#### 2.1 alter table方式创建索引

```mysql
# 添加主键索引
alter table `table_name` add primary key(`column`)
# 添加唯一索引
alter table `table_name` add unique key (`column`)
# 添加普通单列索引
alter table `table_name` add key index_name (`column`)
# 添加普通联合索引
alter table `table_name` add key index_name(`column1`, `column2`,...)
# 添加全文索引
alter table `table_name` add fulltext (`column`)
```



#### 2.2 create table方式创建索引

见本文第一小结.



### 3 索引的触发

Mysql只会对`<、<=、 =、 >=、 >、 between、in`以及`不以通配符%和_开头的like`有效。



示例：

```mysql
# 不会用到索引
select * from bookmarks where description like '%The';
# 会用到索引
select * from bookmarks where description like 'The%';
```



### 4 索引的优缺点

**优点**：提高查询的效率.

**缺点**：

- 降低更新表的速度，比如对表进行INSERT， UPDATE，DELETE。因为在更新表时，MYSQL不仅要保存数据。还要保存一些索引文件。
- 建立索引会占用磁盘空间，若在一个很大的表上创建了多种组合的索引，索引文件将膨胀得很快。



### 5 按物理存储角度分类

从物理存储角度划分,索引可以分为：

- 聚簇索引
- 非聚簇索引



### 6 按数据结构的角度划分

从数据结构角度划分,索引可分为以下几种：

- B+树索引
- Hash索引
- Fulltext索引
- R-Tree索引

可参考另一片笔记:[【索引】高性能MySQL_第三版-创建高性能索引.note](note://2FB3314BE14A4D2CACB7FB44F8B07C31)



------

### 参考资料

PHP与MySQL程序设计 第四版 P532

[MYSQL的索引类型：PRIMARY, INDEX,UNIQUE,FULLTEXT,SPAIAL 有什么区别？各适用于什么场合？](https://blog.csdn.net/u014745069/article/details/80466917)

[面试官:谈谈你对mysql索引的认识？](https://mp.weixin.qq.com/s?__biz=MzIwMDgzMjc3NA==&mid=2247484720&idx=1&sn=7bd7774058e7886eeb3dedb38aa8657a&chksm=96f66759a181ee4f4c177a755c3ac6b6e97fef148bbf4afea8616f4edec33bf6d4f18cda9f69&scene=21#wechat_redirect)           

