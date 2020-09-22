---
title: PHP-使用PDO访问数据库
date: 2020-09-22 18:21:31
tags: ["PHP","PDO"]
categories: ["PHP","PDO"]
---



### 1 前言

一直时用的框架操作数据库，还没有仔细研究过PHP原生的接口，记录一下．

<!--more-->

PDO，PHP Data Objects的简称，是PHP访问数据库的抽象层，可以看作是一个连接数据库的接口，是PHP5之后推荐使用的连接数据库的方式。



### 2.正文
不管是哪种数据库，都可以通过这个接口来连接和操作数据，更换时只需要更换DSN（Data Source Name）即可（有点适配器模式那个意思）；可以有效防止SQL注入（预编译模式）；



### 3 测试数据库

- 连接本地数据库
```bash
$ mysql -uroot -p
Enter password:123456
```
- 数据库: seven
```mysql
# 建库.
create databases seven;
```
- 数据表: users
```mysql
# 建表.
CREATE TABLE `users` (
    `id` int(10) unsigned not null auto_increment COMMENT '自增主键',
    `user_name` varchar(10) not null default '' COMMENT '用户名',
    `age` tinyint(2) not null default 0 COMMENT '年龄',
    PRIMARY KEY (`id`)
) ENGINE InnoDB CHARSET = utf8mb4 COMMENT '用户信息'
```
- 数据：
```mysql
mysql> select * from users;
+----+-----------+-----+
| id | user_name | age |
+----+-----------+-----+
|  4 | Seven     |  18 |
|  5 | Jovry     |  19 |
|  6 | Gordon    |  20 |
+----+-----------+-----+

```

### 4 基本操作
#### 4.1 连接数据库
```php
// 连接数据库.
$dsn = 'mysql:host=127.0.0.1;dbname=seven';
$user = 'root';
$pass = '123456';

try {
    $dbh = new PDO($dsn, $user, $pass);
} catch (PDOException $e) {
    echo "Connection failed: " . $e->getMessage();
    die;
}
```

#### 4.2 查询操作
查询操作有两种方式:
- query(直接查询方式)： 快捷方便,但存在Sql注入的风险.
- prepare + execute(预编译和参数绑定方式)：防止SQL注入,并在插入或修改大量数据时,因为相关sql已经被执行过,所以直接读取即可,速度快.

##### 4.2.1 query方式.
```php
$dbh->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION); // setAttribute用于设置数据库句柄属性,如错误模式等;
$name = 'Seven';
$sql = "select * from users where user_name = " . $dbh->quote($name); // quote是对参数(一般为用户输入)进行转义,防止Sql注入.
foreach ($dbh->query($sql) as $row) {
    echo $row['id'],"\t",$row['user_name'],"\t",$row['age'],"\n";
}
```
执行结果：
```
4       Seven   18
```

##### 4.2.2 prepare + execute方式.
```php
$name = 'Seven';
$sql = 'select * from users where user_name =:name';
$stmt = $dbh->prepare($sql);
$stmt->bindParam(':name', $name, PDO::PARAM_STR);
$stmt->execute();
while($row = $stmt->fetch(PDO::FETCH_ASSOC)) {
    echo $row['id'],"\t",$row['user_name'],"\t",$row['age'],"\n";
}
```
执行结果：
```
4       Seven   18
```

在SQL语句中,通过:name占位符来代替实际变量,然后在bindParam时将该占位符替换成实际的值,通过种种方式避免SQL注入.
$dbh->prepare($sql)返回一个PDOStatement对象,之后都是通过这个对象来进行操作.
PDO::FETCH_ASSOC指定获取方式,将对应结果集中的每一行作为一个由列名索引的数组返回.如果想返回对象，可以使用PDO::FETCH_LAZY或者调用fetchObject方法。


#### 4.3 插入数据
##### 4.3.1 prepare + execute的方式
插入数据也是采用prepare + execute的方式，如果有多个参数，需要进行多次参数绑定。
```php
$name = 'Logan';
$age = 10;
$sql = "insert into users (user_name, age) value (:name, :age)";
$stmt = $dbh->prepare($sql);
$stmt->bindParam(':name', $name, PDO::PARAM_STR);
$stmt->bindParam(':age', $age, PDO::PARAM_INT);
$stmt->execute();
```

执行后数据库中的数据如下：
```mysql
mysql> select * from users;
+----+-----------+-----+
| id | user_name | age |
+----+-----------+-----+
|  4 | Seven     |  18 |
|  5 | Jovry     |  19 |
|  6 | Gordon    |  20 |
| 10 | Logan     |  10 |
+----+-----------+-----+
4 rows in set (0.00 sec)
```

#### 4.4 修改数据
##### 4.4.1 prepare + execute方式
prepare + execute方式. 修改或删除数据，如果有多个参数,需要进行多次数据绑定.
```php
$name = 'Logan';
$age = 15;
$sql = "update users set age = :age where user_name = :name";
$stmt = $dbh->prepare($sql);
$stmt->bindParam(':name', $name, PDO::PARAM_STR);
$stmt->bindParam(':age', $age, PDO::PARAM_INT);
$stmt->execute();
```
执行后数据库中的数据如下：
```mysql
mysql> select * from users;
+----+-----------+-----+
| id | user_name | age |
+----+-----------+-----+
|  4 | Seven     |  18 |
|  5 | Jovry     |  19 |
|  6 | Gordon    |  20 |
| 10 | Logan     |  15 |
+----+-----------+-----+
4 rows in set (0.00 sec)
```

#### 4.5 删除数据
##### 4.5.1 prepare + execute方式
prepare + execute方式. 修改或删除数据，如果有多个参数,需要进行多次数据绑定.
```php
$name = 'Logan';
$sql = "delete from users where user_name = :name";
$stmt = $dbh->prepare($sql);
$stmt->bindParam(':name', $name, PDO::PARAM_STR);
$stmt->execute();
```
执行后数据库中的数据如下：
```mysql
mysql> select * from users;
+----+-----------+-----+
| id | user_name | age |
+----+-----------+-----+
|  4 | Seven     |  18 |
|  5 | Jovry     |  19 |
|  6 | Gordon    |  20 |
+----+-----------+-----+
3 rows in set (0.00 sec)
```

**其实，通过预编译加上参数绑定方式进行增删改查，就是SQL语句不同，其余的都一样。**

#### 4.6 受影响的行 
通过Statement的rowCount方法可以返回上一次操作影响到行数。
```php
$age = 15;
$sql = "select * from users where age > :age";
$stmt = $dbh->prepare($sql);
$stmt->bindParam(':age', $age, PDO::PARAM_INT);
$stmt->execute();
echo $stmt->rowCount();
```
执行脚本输出如下：
```
3
```

### 5 事务支持
PDO也支持MySQL事务（transaction），包括start、commit和rollback等。
```php
try {
    $dbh->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
    $dbh->beginTransaction();  // 开始事务.
    $sql  = "insert into users (user_name, age) values (:name, :age)";
    $stmt = $dbh->prepare($sql);
    $stmt->execute(array(':name' => 'Natasha', ':age' => 23));
    $id = 1;
    $sql = 'delete from users where id = :id';
    $stmt = $dbh->prepare($sql);
    $stmt->bindParam(':id', $id, PDO::PARAM_INT);
    $stmt->execute();
    $dbh->commit(); // 提交事务.
} catch(PDOException $e) {
    $dbh->rollBack(); // 回滚事务.
    echo "Error occurred when execute SQL: ", $sql, "<br>";
    echo "Error info:<br>", $e->getMessage();
}
```
执行后数据库中的数据如下：
```mysql
mysql> select * from users;
+----+-----------+-----+
| id | user_name | age |
+----+-----------+-----+
|  4 | Seven     |  18 |
|  6 | Gordon    |  20 |
| 11 | Natasha   |  23 |
+----+-----------+-----+
3 rows in set (0.00 sec)
```

### 6 错误处理
PDO有3个错误提示等级，分别为`POD::ERRMODE_SILENT`（默认）、`POD::ERRMODE_WARNING`和`POD::ERRMODE_EXCEPTION`。

#### 6.1 POD::ERRMODE_SILENT
不显示错误信息，会设置一个错误码，需要检查错误码来判断是否发生错误。
```php
$dbh->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_SILENT); // 问题：默认为POD::ERRMODE_SILENT，但是当未设置错误级别时，php抛出了Fatal error？？？？
$name = 'Seven';
$sql = 'SELECT * FROM user WHERE user_name =:name';   // 数据表名有误
$stmt = $dbh->prepare($sql);
$stmt->bindParam(':name', $name, PDO::PARAM_STR);
$stmt->execute();
$code = $stmt->errorCode();
//var_dump($code);echo '<br>';   // 输出错误码，string(5) "42S02"
if (empty($code)) {
    while ($row = $stmt->fetchObject()) {
        echo $row['id'],"\t",$row['name'],"\t",$row['score'],"<br>";
    }
}else{
    echo 'Error occurred when execut SQL ', $sql, '<br>';
    echo 'ErrorInfo:<br><pre>';
    var_dump($stmt->errorInfo());
}
```
执行脚本结果：
```
Error occurred when execut SQL SELECT * FROM user WHERE user_name =:name<br>ErrorInfo:<br><pre>array(3) {
  [0]=>
  string(5) "42S02"
  [1]=>
  int(1146)
  [2]=>
  string(32) "Table 'seven.user' doesn't exist"
}
```
当未设置错误级别时，输出结果为：
```
PHP Fatal error:  Uncaught exception 'PDOException' with message 'SQLSTATE[42S02]: Base table or view not found: 1146 Table 'seven.user' doesn't exist' in /home/cdyf/tutorial/PDO/Pdo_test.php:119
Stack trace:
#0 /home/cdyf/tutorial/PDO/Pdo_test.php(119): PDOStatement->execute()
#1 {main}
  thrown in /home/cdyf/tutorial/PDO/Pdo_test.php on line 119
```

注意:**这里的错误是Statement级别的,不是PHP级别,所以需要使用$stmt->errorInfo()来获取错误信息**.

#### 6.2 PDO::ERRMODE_WARNING
该模式下,出错会产生一个PHP的警告(warning), 同时会设置Statement级别和errorinfo.
```php
$dbh->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_WARNING);
$name = "Seven";
$sql = 'SELECT * FROM user WHERE user_name =:name';
$stmt = $dbh->prepare($sql);
$stmt->bindParam(':name', $name, PDO::PARAM_STR);
$stmt->execute();
$code = $stmt->errorCode();
// var_dump($code);echo '<br>';
if (empty($code)) {
    while ($row = $stmt->fetchObject()) {
        echo $row['id'],"\t",$row['name'],"\t",$row['score'],"<br>";
    }
}else{
    echo 'Error occurred when execut SQL ', $sql, '<br>';
    echo 'ErrorInfo:<br><pre>';
    var_dump($stmt->errorInfo());
}
```
执行脚本结果：
```
PHP Warning:  PDOStatement::execute(): SQLSTATE[42S02]: Base table or view not found: 1146 Table 'seven.user' doesn't exist in /home/cdyf/tutorial/PDO/Pdo_test.php on line 157
Error occurred when execut SQL SELECT * FROM user WHERE user_name =:name<br>ErrorInfo:<br><pre>array(3) {
  [0]=>
  string(5) "42S02"
  [1]=>
  int(1146)
  [2]=>
  string(32) "Table 'seven.user' doesn't exist"
}
```
**与SILENT模式相比，多了一个Warning.**

#### 6.3 PDO:: ERRMODE_EXCEPTION
在这个模式下，出错会产生一个PHP的错误（PDOException），同时也会设置Statement级别的errorinfo。不过这个时候，我们就可以通过try{...}catch{...}的方式来捕获产生的异常。
```php
$dbh->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
$name = 'Seven';
try {
    $sql = 'SELECT * FROM user WHERE user_name =:name';
    $stmt = $dbh->prepare($sql);
    $stmt->bindParam(':name', $name, PDO::PARAM_STR);
    $stmt->execute();
    while ($row = $stmt->fetchObject()) {
        echo $row['id'],"\t",$row['name'],"\t",$row['score'],"<br>";
    }
}catch(PDOException $e) {
    echo 'PDO Exception Caught.' . "\n";
    echo 'Error occurred with executed SQL: '.$sql."\n";
    echo 'Error: ' . $e->getMessage(). "\n";
    echo 'Code: '. $e->getCode(). "\n";
    echo 'File: '. $e->getFile(). "\n";
    echo 'Line: '. $e->getLine(). "\n";
    echo 'Trace: '. $e->getTraceAsString(). "\n";
}
```
执行脚本结果：
```
PDO Exception Caught.
Error occurred with executed SQL: SELECT * FROM user WHERE user_name =:name
Error: SQLSTATE[42S02]: Base table or view not found: 1146 Table 'seven.user' doesn't exist
Code: 42S02
File: /home/cdyf/tutorial/PDO/Pdo_test.php
Line: 196
Trace: #0 /home/cdyf/tutorial/PDO/Pdo_test.php(196): PDOStatement->execute()
#1 {main}
```
通过try{...}catch{...}的方式，可以捕获execute时产生的异常，可以跟踪到文件和行数级别。

### 7 防止SQL注入
- SQL注入就是攻击者在用户输入的地方输入sql语句从而导致该语句在数据库中执行.
- 避免SQL注入的准则: 在外部署据到达数据库之前线对他进行转义.

#### 7.1 query方式防止sql注入方法
query方式不能自动转义,需要手动调用$hdb->quote方法来转.

#### 7.2 预编译方法避免sql注入原理
- 其原理如下:   
当调用prepare()时，查询语句已经发送给了数据库服务器，此时只有占位符 发送过去，没有用户提交的数据；当调用到execute()时，用户提交过来的值才会传送给数据库，它们是分开传送的，所以不会产生SQL注入的风险。