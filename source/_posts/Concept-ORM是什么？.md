---
title: Concept-什么是ORM？
date: 2020-09-25 22:27:39
tags: ["Concept","ORM"]
categories: ["Concept"]
---

### 1 什么是ORM?

ORM（Object Relation Mapping，对象关系映射）是一种程序设计技术，通过面向对象编程语言的语法，完成关系型数据库的操作．

<!--more-->

ORM把数据库映射成对象：

```
数据库的表（table）   --->  类（class）
记录（record，行记录） --->  对象（object）
字段（filed）        --->   对象的属性（attribute）
```



示例所示：通常对于数据库，采用SQL方式查询如下：

```mysql
select * from users where uid = 123;
```



ORM通过使用我们自行选择的语言而非SQL与数据库进行交互：

```
var orm = require（'generic-orm-libarry'）;
var user = orm（“users”）．where（{uid：123}）;
```



### 2 ORM的优缺点

优点：

- 数据模型都在一个地方定义，更容易更新和维护，也利于代码重用；
- ORM有现成的工具，许多高级功能可以自动完成，例如数据消毒、事务、连接池、迁移、流等其他功能的支持；
- 当使用MVC架构时，ORM就是现成的Model，使最终代码更加清晰；
- 基于ORM的业务代码更简单，代码量少，语义清晰，容易理解；
- 它抽象了数据库系统，因此可以更为方便地切换数据库类型，如从MySQL切换到PostgreSQL；
- 使许多查询的性能比自行编写SQL语句的性能更好；
- 对于没有DBA的小型团队中，使用ORM将极大简化数据层的工作。



缺点：

- ORM库不是轻量级工具，将增加学习ORM的成本；
- 对于复杂的查询，ORM可能无法表示，或性能不如原生的SQL;
- ORM的初始配置可能较为复杂；
- 作为开发人员，重要的是要了解其原理，而ORM抽象掉了数据层，可能导致开发者对底层原理理解不到位，也无法定制一些特殊的SQL;



### 3 关系类型

表与表之间的关系，分为三种：

- `一对一`：一种对象与另一种对象是一一对应关系；比如一个顾客只能对应一张入场票；
- `一对多`：一种对象可以属于另一个对象的多个实例，比如一张唱片包含多首歌；
- `多对多`：两种对象彼此都是＂一对多＂关系，比如一张唱片包含多首歌，同时一首歌可以属于多张唱片．



### 4 ORM的实现

ORM中有两种常用的实现模式：`Active Record`和`Data Mapper`



#### 4.1 Active Record

Active Record模式的ORM将对象映射到数据库行．



如下示例所示：创建一个User对象，设置username，然后将该对象保存到数据库中．将User对象映射到users表中的一行．

```php
$user = new User;  
$user->username = ‘philipbrown’;  
$user->save();  
```



Active Record模式的优点：

- 可以简单的调用`save()`方法来更新数据库．
- 每个模型对象都继承自基础Active Record对象，因此可以访问所有与持久性有关的方法．
- Active Record实现非常直观，容易上手．



#### 4.2 Data Mapper

Active Record和Data Mapper模式之间的最大区别是，Data Mapper模式将对象与持久层完全分开．



如下示例所示：Data Mapper使用一个完全不同的服务，如`EntityManager`进行数据持久化到数据库．

```php
$user = new User;  
$user->username = ‘philipbrown’;  
EntityManager::persist($user);  
```



Data Mapper模式的优点：

- 对象不需要了解他们如何存储数据库，因此对象将更加轻便，因为它们不必继承完整的ORM．
- 与数据库进行交互的过程更加严格，不能在代码中调用`save()`方法．



### 5 常见ORM框架

各种语言常见的ORM框架【*参考资料3*】

- JAVA：ActiveJDBC、Hibernate、MyBatis等
- PHP：Doctrine、Yii、Laravel等



------

### 参考资料

1. [What is an ORM and Why You Should Use it](https://blog.bitsrc.io/what-is-an-orm-and-why-you-should-use-it-b2b6f75f5e2a)
2. [对象关系映射](https://zh.wikipedia.org/wiki/%E5%AF%B9%E8%B1%A1%E5%85%B3%E7%B3%BB%E6%98%A0%E5%B0%84)
3. [List of object-relational mapping software](https://en.wikipedia.org/wiki/List_of_object-relational_mapping_software)
4. [Active Record和Data Mapper有什么区别？](https://culttt.com/2014/06/18/whats-difference-active-record-data-mapper/)

