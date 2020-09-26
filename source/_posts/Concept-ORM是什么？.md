---
title: Concept-什么是ORM？
date: 2020-09-25 22:27:39
tags: ["Concept","ORM"]
categories: ["Concept"]
---

### 1 什么是ORM?

ORM（Object Relation Mapping，对象关系映射）是一种程序设计技术，用于实现面向对象编程语言里不同类型系统的资料之间的转换，从效果上说，它其实是创建了一个在可编程语言里使用的“虚拟对象数据库”。

通常对于数据库，采用SQL方式查询，如下示例所示：

```mysql
select * from users where uid = 123;
```



ORM使我们使用我们自行选择的语言而非SQL与数据库进行交互，如下示例：

```
var orm = require（'generic-orm-libarry'）;
var user = orm（“users”）。where（{uid：123}）;
```



### 2 ORM的优点

- 可以使用一种更为熟悉的语言对数据库进行操作；
- 它抽象了数据库系统，因此可以更为方便地切换数据库类型，如从MySQL切换到PostgreSQL；
- 根据ORM，可以获取许多高级功能，例如对事物、连接池、迁移、流等其他功能的支持；
- 使许多查询的性能比自行编写SQL语句的性能更好；
- 对于没有DBA的小型团队中，使用ORM将极大简化数据层的工作。



### 3 ORM的缺点

- 限制了SQL编写，对于熟悉SQL的人员，可以通过自行编写查询来获得性能更高的查询；
- 增加学习ORM的成本；
- ORM的初始配置可能较为复杂；
- 作为开发人员，重要的是要了解其原理，使用ORM可能导致对底层原理理解不到位。



### 4 常见ORM框架

各种语言常见的ORM框架【*参考资料3*】

- JAVA：ActiveJDBC、Hibernate、MyBatis等
- PHP：Doctrine、Yii、Laravel等



------

### 参考资料

1. [What is an ORM and Why You Should Use it](https://blog.bitsrc.io/what-is-an-orm-and-why-you-should-use-it-b2b6f75f5e2a)
2.  [对象关系映射](https://zh.wikipedia.org/wiki/%E5%AF%B9%E8%B1%A1%E5%85%B3%E7%B3%BB%E6%98%A0%E5%B0%84) 
3.  [List of object-relational mapping software](https://en.wikipedia.org/wiki/List_of_object-relational_mapping_software)

