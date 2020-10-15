---
title: PHP-Laravel简介及安装
date: 2020-10-05 16:43:13
tags: ["PHP","Laravel"]
categories: ["PHP","Laravel"]
---

1 简介

Laravel是一套简洁、优雅的PHP Web开发框架(PHP Web Framework)。

<!--more-->

2 安装

2.1 服务器要求

确保服务器满足版本要求且已安装以下扩展：

扩展：

- BCMath
- Ctype
- Fileinfo
- JSON
- Mbstring
- OpenSSL
- PDO
- Tokenizer
- XML



2.2 安装Laravel

Laravel 使用 [Composer](https://getcomposer.org/) 来管理项目依赖。因此，在使用 Laravel 之前，请确保您的机器上已经安装了 Composer。

2.2.1 通过Laravel安装器

*注意：使用安装器安装为最新版本，当前安装的版本为laravel 8.x*

2.2.1.1 使用composer安装Laravel安装器

```
composer global require laravel/installer
```



2.2.1.2 修改系统环境变量

```txt ~/.bashrc
export LARAVEL_HOME=/root/.composer/vendor/bin
export PATH=$LARAVEL_HOME:$PATH
```



2.2.1.3 执行以下命令使其生效

```
source ~/.bashrc
```



2.2.1.4 创建Laravel项目

通过`laravel new`命令创建Laravel项目。

示例：通过以下命令将创建一个名为blog的目录，并安装号Laravel所有依赖项。

```
laravel new blog
```



2.2.2 通过Composer创建项目

通过create-project命令来安装Laravel：

```
composer create-project --prefer-dist laravel/laravel blog <版本>
```



示例：由于我本机php版本为7.1.3，查看支持的Laravel版本，对应的应该为5.6

```
composer create-project --prefer-dist laravel/laravel blog 5.6.*
```



