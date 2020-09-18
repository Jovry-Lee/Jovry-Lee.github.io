---
title: Docker-容器
date: 2020-09-16 17:26:46
tags: ["Docker"]
categories: ["Docker"]
---



### 1 概述

容器是独立运行的一个或一组应用，以及它们的运行态环境。对应的，虚拟机可以理解为模拟运行的一整套操作系统（提供了运行态环境和其他系统环境）和跑在上面的应用。

<!--more-->



### 2 启动容器

`容器的启动有以下两种情况`：

- ①、基于镜像新建一个容器并启动
- ②、将在终止状态的容器重新启动



#### 2.1 基于镜像新建容器并启动

使用docker run命令新建并启动一个容器的，其语法格式为：

```
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```

常用OPTIONS：

- -t（--tty）：Docker分配一个伪终端（pseudo-tty）并绑定到容器的标准输入上．
- -i（--interactive）：让容器的标准输入保持打开（交互模式）．eg：以交互式方式打开启动一个容器：`docker run -t -i ubuntu:18.04 /bin/bash`
- -d（--detach）：后台运行容器，并返回容器ID．eg：`docker run -d ubuntu:18.04 /bin/sh -c "while true; do echo hello world; sleep 1; done"`



##### 2.1.1 Docker在后台运行的标准操作

当利用 `docker run` 来创建容器时，Docker 在后台运行的标准操作包括：

- 检查本地是否存在指定的镜像，不存在就从公有仓库下载
- 利用镜像创建并启动一个容器
- 分配一个文件系统，并在只读的镜像层外面挂载一层可读写层
- 从宿主主机配置的网桥接口中桥接一个虚拟接口到容器中去
- 从地址池配置一个 ip 地址给容器
- 执行用户指定的应用程序
- 执行完毕后容器被终止



#### 2.2 启动已终止的容器

使用 `docker container start` 命令，直接将一个已经终止的容器启动运行．其语法格式为：

```
docker container start [OPTIONS] CONTAINER [CONTAINER...]
```

常用OPTIONS：

- -a（--attach）：指定标准输入输出内容类型，可选 `STDOUT`/`STDERR` ；



### 3 终止容器

使用 `docker container stop` 来终止一个或多个运行中的容器，其语法格式为：

```
docker container stop [OPTIONS] CONTAINER [CONTAINER...]
```

OPTIONS：

- -t（--time）：终止容器前的等待时间，单位秒（默认10s）．



### 4 重启一个容器

使用 `docker container restart` 来重启一个或多个运行中的容器，其语法格式为：

```
docker container restart [OPTIONS] CONTAINER [CONTAINER...]
```

OPTIONS：

- -t（--time）：终止容器前的等待时间，单位秒（默认10s）．



注： `docker container restart` 命令会将一个运行态的容器终止，然后再重新启动它。



### 5 进入容器

在使用 `-d` 参数启动容器后会进入后台。Docker提供了两种方式进入容器进行操作，分别是：

- `docker attach`（不推荐）
- `docker exec`

#### 5.1 attach 命令

attach命令的语法格式如下：

```
docker attach [OPTIONS] CONTAINER
```

注：<u>如果从这个 stdin 中 exit，会导致容器的停止。</u>因此，不推荐使用attach命令．



#### 5.2 exec命令

exec命令的语法格式如下：

```
docker exec [OPTIONS] CONTAINER COMMAND [ARG...]
```

常用OPTIONS：

- -i（--interactive）：保持标准输入状态．
- -t（--tty）：分配一个伪终端，与`-i`联合使用可实现交互式模式．eg：`docker exec -i 69d1 bash`



### 6 删除容器

使用 `docker container rm` 来删除一个或多个容器．其语法格式为：

```
docker container rm [OPTIONS] CONTAINER [CONTAINER...]
```

常用OPTIONS：

- -f（--force）：强制删除一个容器（删除一个运行中的容器）．



#### 6.1 删除所有处于终止状态的容器

docker container prune用于删除所有处于终止状态的容器，其语法格式如下：

```
docker container prune [OPTIONS]
```

常用OPTIONS:

- --filter：提供过滤值．



### 7 容器管理

使用docker container对容器进行管理，其语法格式为：

```
docker container COMMAND
```

常用的COMMAND：

- start：启动一个或多个终止状态的容器．
- stop：终止一个或多个运行中的容器．
- restart：重启一个或多个运行中的容器．
- rm：删除一个或多个容器．
- prune：删除所有处于终止状态的容器．
- ls：查看容器信息．
- logs：获取容器的输出信息（获取后台运行的容器的输出信息）．格式：`docker container logs [OPTIONS] CONTAINER`

