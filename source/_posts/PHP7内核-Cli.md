---
title: PHP7内核-Cli
date: 2020-08-21 18:06:04
tags: ["PHP"]
categories: ["PHP"]
---



#### 1 简介
Cli（Command line Interface），命令行接口，用于在命令行下执行PHP脚本，类似于Shell那样，是执行PHP脚本最简便的一种方式。

<!--more-->

Cli模式通过执行变异的PHP二进制程序即可启动，它定义了很多命令行参数，不同的参数对应不同的处理，例如：
- 获取cli的参数的帮助文档：`-h参数`
```
php -h
```
- 执行PHP脚本文件：`php  脚本名.php`
```
php script.php
```
- 直接执行PHP代码： `-r参数`(代码需用引号括起来)
```
php -r "phpinfo();"
```
- 输出PHP版本：`-v参数`
```
php -v
```
- 输出已安装的扩展：`-m参数`
```php
php -m
```
- 交互模式运行PHP：`-a参数`
```php
$ php -a
Interactive mode enabled


php > $a = 1;
php > $b = 2;
php > echo $a + $b;
3
```
#### 2 执行流程
<u>Cli是单进程模式，处理完请求后就直接关闭了</u>，**生命周期**为：

- ①、`module startup`
- ②、`request startup`
- ③、`execute script`
- ④、`request shutdown`
- ⑤、`module shurdown`



处理的关键过程为：
main()->php_cli_startup()->do_cli()->php_module_shutdown()

*注：若是查询系统信息之类的请求，如：-v、-m、-i之类的，则不需要经历PHP请求的生命周期。*



#### 3 Cli模式下获取要运行的PHP代码

##### 3.1 让PHP运行指定文件
以下两种方法（使用或不使用 -f 参数）都能够运行给定的 my_script.php 文件。可以选择任何文件来运行，**指定的PHP脚本并非必须要以 .php 为扩展名**，它们可以有任意的文件名和扩展名。
```
php my_script.php

php -f my_script.php
```
##### 3.2 载明两行直接运行PHP代码
使用-r参数引用要执行的代码, 其中代码需要`使用引号括起来`。  
*注：此种方式下php代码不能添加开始和结束标记符。*

```
php -r 'print_r(get_defined_constants());'
```

##### 3.3 通过标准输入（stdin）提供需要运行的php代码
使用该方式，可以动态地生成PHP代码并通过命令行运行这些代码
```
$ some_application | some_filter | php | sort -u >final_output.txt
```

##### 3.5 将php脚本作为shell脚本使用
写一个php脚本，并在第一行以`#！/usr/bin/php`开头，在其后加上以 PHP 开始和结尾标记符包含的正常的 PHP 代码，然后为该文件设置正确的运行属性（例如：`chmod +x test`）。该方法可以使得该文件能够像Shell脚本或 Perl 脚本一样被直接执行。

```
#!/usr/bin/php
<?php
    var_dump($argv);
?>
```

假设文件名为test，并放置在当前目录下：
```
$ chomod +x test
$ ./test --foo
array(2) {
  [0] =>
  string(6) "./test"
  [1] =>
  string(5) "--foo"
}

```

#### 4 内置web服务器
从PHP5.4.0开始，Cli SAPI提供了一个内置web服务器，这个内置的Web服务器主要用于本地开发使用，不可用于线上产品环境。  
详见：[内置Web Server](https://www.php.net/manual/zh/features.commandline.webserver.php)