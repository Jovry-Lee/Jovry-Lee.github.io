---
title: Protobuf简介及安装
date: 2020-08-25 16:14:48
tags: ["Protobuf"]
categories: ["Protobuf"]
---

#### 1 简介

Protobuf是谷歌的开源序列化协议框架，结构类似于XML，JSON这种，显著的特点是二进制的，效率高，主要用于通信协议和数据存储等方面，算是一种结构化数据的表示方法。

<!--more-->

#### 2 安装方法

##### 2.1 Ubuntu下安装

- 要编译安装protobuf，首先需要安装以下必要的工具.

  安装方法：

```bash
sudo apt-get install autoconf automake libtool curl make g++ unzip
```

- 获取源文件

  - 方法一：直接下载发行版

    [下载地址](https://github.com/protocolbuffers/protobuf/releases/tag/v3.11.4)

  - 方法二：通过git clone下载 

    ```bash
    git clone https://github.com/protocolbuffers/protobuf.git cd protobuf git submodule update --init --recursive ./autogen.sh
    ```

  - 编译安装C++ Protocal Buffer运行时和Protocal Buffer编译器（protoc）

    ```bash
    ./configure make make check sudo make install sudo ldconfig # refresh shared library cache.
    ```

    *注：默认安装位置为：*

    ```bash
    /usr/local
    ```

    可通过以下命令指定安装路径

    ```bash
    ./configure --prefix=/usr
    ```

  - 查看是否安装成功
  
    ```bash
    protoc --version
    ```
  


#### 参考资料

1. [protocolbuffers/protobuf](https://github.com/protocolbuffers/protobuf/tree/master/php)

2. [Protobuf 小试牛刀](https://www.cnblogs.com/52fhy/p/11106670.html)

3. [protobuf简单介绍和ubuntu 16.04环境下安装](https://blog.csdn.net/kdchxue/article/details/81046192)