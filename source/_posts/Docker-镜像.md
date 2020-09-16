---
title: Docker-镜像的操作
date: 2020-09-16 17:30:51
tags: ["Docker"]
categories: ["Docker"]
---



### 1 概述

镜像是Docker的三大核心之一．Docker 运行容器前需要本地存在对应的镜像，如果本地不存在该镜像，Docker 会从镜像仓库下载该镜像。



### 2 获取镜像

 [Docker Hub](https://hub.docker.com/search?q=&type=image) 上有大量的高质量的镜像可以用，通过`docker pull`命令可以从中获取．

命令格式：

```
docker pull [选项] [Docker Registry 地址[:端口号]/]仓库名[:标签]
```

- Docker 镜像仓库地址：地址的格式一般是 `<域名/IP>[:端口号]`。默认地址是 Docker Hub(docker.io)。
- 仓库名：如之前所说，这里的仓库名是两段式名称，即 `<用户名>/<软件名>`。对于 Docker Hub，如果不给出用户名，则默认为 `library`，也就是官方镜像。



示例：`sudo docker pull ubuntu:18.04`



### 3 列出镜像

列出已经下载下来的镜像

```
docker image ls [选项] [仓库名][:标签]
```

常用选项：

- -a：显示包括中间层镜像在内的所有镜像；
- -f：过滤出指定镜像；eg：`image ls -f since=ubuntu:18.04`
- -q：列出镜像ID；
- --format：列出指定内容的镜像；eg：`docker image ls --format "{{.ID}}: {{.Repository}}"`



### 4 查看镜像体积

查看镜像、容器、数据卷所占用的空间命令：

```
docker system df
```

注：*`docker image ls`列出的镜像体积总和并非是所有镜像实际硬盘消耗。由于Docker镜像是多层存储结构，并且可以继承、复用，因此不同镜像可能会因为使用相同的基础镜像，从而拥有共同的层。Docker使用Union FS，相同的层只需要保存一份即可，因此实际镜像磁盘占用空间很可能比这个列表镜像大小的总和要小很多。*



### 5 删除本地镜像

删除本地镜像命令：

```bash
docker image rm [选项] <镜像1> [<镜像2>...]
```

其中镜像可以是：

- 镜像短ID
- 镜像长ID
- 镜像名
- 镜像摘要

注：*`docker image ls`默认列出的就已经是短ID了，一般取前3个字符以上，就足够区分与别的镜像了。*



##### 5.1 批量删除镜像

使用`docker image ls`命令配合使用`docker image rm`，实现成批删除镜像。



示例：

```
1. 删除所有仓库名为redis的镜像
docker image rm $(docker image ls -q redis)

2.删除所有在mongo:3.2之前的镜像
docker image rm $(docker image ls -q -f before=mongo:3.2)
```

