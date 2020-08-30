---
title: Ubuntu16.04-Nodejs安装
date: 2020-08-21 15:28:42
tags: ["Ubuntu","Config","Node.js"]
categories: ["Ubuntu", "Config"]
---



#### 下载 

- ①、下载安装包

  在ubuntu环境下，前往[nodejs官网](https://nodejs.org/en/)，nodejs官网能自动检测自己的系统版本，推荐出合适的nodejs版本。

  <!--more-->

- ②、解压安装包

```bash
$ tar xvJf node-v12.18.3-linux-x64.tar.xz
```

- ③、拷贝安装文件到指定路径

```bash
sudo cp -r node-v12.18.3-linux-x64 /usr/local/
```

#### 配置

- 修改~/.bashrc文件

```
#set Node.js
export NODE_HOME=/usr/local/node-v12.18.3-linux-x64
export PATH=$NODE_HOME/bin:$PATH
```

- 执行source命令

```
$ source ~/.bashrc
```

- 3、添加软连接

```
ln -s /usr/local/node-v12.18.3-linux-x64/bin/node /usr/bin/node
ln -s /usr/local/node-v12.18.3-linux-x64/bin/npm /usr/bin/npm
```

- 4、查看node版本

```
$ node -v
v12.18.3
$ npm -v
6.14.6
```

能查看版本,即表示安装成功.

#### 参考资料

[Ubuntu 16.04环境下nodejs的安装和配置](https://blog.csdn.net/qq_36272282/article/details/88887360)