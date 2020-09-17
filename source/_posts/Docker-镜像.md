---
title: Docker-镜像的操作
date: 2020-09-16 17:30:51
tags: ["Docker"]
categories: ["Docker"]
---



#### 1 概述

镜像是Docker的三大核心之一．Docker 运行容器前需要本地存在对应的镜像，如果本地不存在该镜像，Docker 会从镜像仓库下载该镜像。

<!--more-->

#### 2 获取镜像

 [Docker Hub](https://hub.docker.com/search?q=&type=image) 上有大量的高质量的镜像可以用，通过`docker pull`命令可以从中获取．

命令格式：

```
docker pull [选项] [Docker Registry 地址[:端口号]/]仓库名[:标签]
```

- Docker 镜像仓库地址：地址的格式一般是 `<域名/IP>[:端口号]`。默认地址是 Docker Hub(docker.io)。
- 仓库名：如之前所说，这里的仓库名是两段式名称，即 `<用户名>/<软件名>`。对于 Docker Hub，如果不给出用户名，则默认为 `library`，也就是官方镜像。



示例：`sudo docker pull ubuntu:18.04`



#### 3 列出镜像

列出已经下载下来的镜像

```
docker image ls [OPTIONS] [REPOSITORY[:TAG]]
```

常用OPTIONS：

- `-a（--all）`：显示包括中间层镜像在内的所有镜像．
- `-f（--filter）`：过滤出指定镜像．eg：`image ls -f since=ubuntu:18.04`
  - `--format`：使用[Go模板语法](https://gohugo.io/templates/introduction/)列出指定内容的镜像．eg：`docker image ls --format "{{.ID}}: {{.Repository}}"`
- `-q`：列出镜像ID．eg：`docker image ls -q`



#### 4 查看镜像体积

查看镜像、容器、数据卷所占用的空间命令：

```
docker system df
```

注：*`docker image ls`列出的镜像体积总和并非是所有镜像实际硬盘消耗。由于Docker镜像是多层存储结构，并且可以继承、复用，因此不同镜像可能会因为使用相同的基础镜像，从而拥有共同的层。Docker使用Union FS，相同的层只需要保存一份即可，因此实际镜像磁盘占用空间很可能比这个列表镜像大小的总和要小很多。*



#### 5 删除本地镜像

删除本地镜像命令：

```bash
docker image rm [OPTIONS] IMAGE [IMAGE...]
```

删除命令的别名还包括：`rm`，`rmi`，`remove`．

其中镜像可以是：

- 镜像短ID
- 镜像长ID
- 镜像名
- 镜像摘要

注：*`docker image ls`默认列出的就已经是短ID了，一般取前3个字符以上，就足够区分与别的镜像了。*



常用OPTIONS:

- `-f（--force）`：强制删除镜像．



##### 5.1 批量删除镜像

使用`docker image ls`命令配合使用`docker image rm`，实现成批删除镜像。



示例：

```
1. 删除所有仓库名为redis的镜像
docker image rm $(docker image ls -q redis)

2.删除所有在mongo:3.2之前的镜像
docker image rm $(docker image ls -q -f before=mongo:3.2)
```



#### 6 构建镜像

使用docker build命令进行镜像构建．其格式为：

```
docker build [OPTIONS] PATH | URL | -
```

常用OPTIONS:

- -t：以name:tag格式对镜像命名．eg：`docker build -t nginx:v3 .`
- -f：指定Dockerfile文件（默认为path/Dockerfile）
- --build-arg：设置编译时的变量．eg：`docker build --build-arg name="Jovry"`



##### 6.1 docker build的工作原理

Docker在运行时分为Docker引擎（也就是服务端守护进程）和客户端工具。Docker的引擎提供了一组REST API，被称为 [Docker Remote API](https://docs.docker.com/develop/sdk/)，而如 `docker` 命令这样的客户端工具，则是通过这组 API 与 Docker 引擎交互，从而完成各种功能。

因此，虽然表面上我们好像是在本机执行各种 `docker` 功能，但实际上，一切都是使用的远程调用形式在服务端（Docker 引擎）完成。



##### 6.2 构建上下文

通过docker build的工作原理可知，docker运行是采用的C/S架构，为了解决让服务端获取本地文件的问题，因此引入了上下文的概念，在构建的时候，指定构建镜像的上下文的路径．

`docker build` 命令拿到这个路径后，会将路径下的所有内容打包，然后上传给 Docker 引擎。这样 Docker 引擎收到这个上下文包后，展开就会获得构建镜像所需的一切文件。



在`docker build -t nginx:v3 .`示例中，`．`即表示上下文目录．  `docker build` 命令会将该目录下的内容打包交给 Docker 引擎以帮助构建镜像。



<u>推荐做法</u>：将 `Dockerfile` 置于一个空目录下，或者项目根目录下。如果该目录下没有所需文件，那么应该把所需文件复制一份过来。如果目录下有些东西确实不希望构建时传给 Docker 引擎，那么可以用 `.gitignore` 一样的语法写一个 `.dockerignore`，该文件是用于剔除不需要作为上下文传递给 Docker 引擎的。



##### 6.3 其他构建方式

- 使用Git Repo进行构建：`docker build`支持从URL进行构建，比如可以直接从Git repo中构建．eg：`docker build -t hello-world https://github.com/docker-library/hello-world.git#master:amd64/hello-world`
- 用给定的tar压缩包构建：Docker 引擎会下载这个包，并自动解压缩，以其作为上下文，开始构建。eg：`docker build http://server/context.tar.gz`
- 从标准输入中读取Dockerfile进行构建： `docker build - < Dockerfile` 或 `cat Dockerfile | docker build -`
- 从标准输入中读取上下文压缩包进行构建： `docker build - < context.tar.gz`              



#### ７ 定制镜像

`Docker镜像定制有哪些方式`？

- ①、使用commit命令定制（不推荐）
- ②、使用Dockerfile定制镜像



##### 7.1 使用commit命令定制

镜像是多层存储，每一层是在前一层的基础上进行的修改；而容器同样也是多层存储，是在以镜像为基础层，在其基础上加一层作为容器运行时的存储层。

当运行一个容器的时候（如果不使用卷的话），我们做的`任何文件修改都会被记录于容器存储层里`。而 Docker 提供了一个 `docker commit` 命令，可以将容器的存储层保存下来成为镜像。换句话说，就是在原有镜像的基础上，再叠加上容器的存储层，并构成新的镜像。以后我们运行这个新镜像的时候，就会拥有原有容器最后的文件变化。



###### 7.1.1 查看容器改动

Docker提供了一个命令查看修改的容器文件，如下：

```
docker diff CONTAINER
```



###### 7.1.2 保存容器的修改

`docker commit` 的语法格式为：

```
docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]
```



OPTIONS：

- -a（--author）：作者名
- -c（ --change）使用Dockerfile结构创建镜像．
- -m（--message） ：注释信息．
- -p（--pause）：提交期间暂停容器（默认为true）



###### 7.1.3 查看镜像内的历史记录

docker history的语法格式为：

```
docker history [OPTIONS] IMAGE
```



###### 7.1.4 为什么不推荐使用docker commit定制镜像

- 可能导致很多无关的文件被改动或者添加导致镜像变得非常臃肿．
- 直接操作的镜像也被称为黑箱镜像，除了制作人知道修改过程，其他人无法得知，维护非常艰难．
- 由于镜像为分层结构，每次修改都会让镜像更臃肿一次．



##### 7.2　Dockerfile定制镜像

Dockerfile 是一个文本文件，其内包含了一条条的 `指令(Instruction)`，每一条指令构建一层，因此每一条指令的内容，就是描述该层应当如何构建。

[Dockerfile相关指令](https://jovry-lee.github.io/2020/09/17/Docker-Dockerfile%E6%8C%87%E4%BB%A4)









