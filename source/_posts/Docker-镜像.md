---
title: Docker-镜像
date: 2020-09-16 17:30:51
tags: ["Docker"]
categories: ["Docker"]
---



1 概述

镜像是Docker的三大核心之一．Docker 运行容器前需要本地存在对应的镜像，如果本地不存在该镜像，Docker 会从镜像仓库下载该镜像。



2 获取镜像

 [Docker Hub](https://hub.docker.com/search?q=&type=image) 上有大量的高质量的镜像可以用，通过`docker pull`命令可以从中获取．

命令格式：

```
docker pull [选项] [Docker Registry 地址[:端口号]/]仓库名[:标签]
```

- Docker 镜像仓库地址：地址的格式一般是 `<域名/IP>[:端口号]`。默认地址是 Docker Hub(docker.io)。
- 仓库名：如之前所说，这里的仓库名是两段式名称，即 `<用户名>/<软件名>`。对于 Docker Hub，如果不给出用户名，则默认为 `library`，也就是官方镜像。



示例：

