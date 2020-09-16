---
title: Docker-错误处理
date: 2020-09-16 17:57:09
tags: ["Docker"]
categories: ["Docker"]
---



记录一些遇见的错误，及解决方法．

<!--more-->

1 删除镜像错误

1.1 image is being used by stopped container <xxx>

报错信息：

```bash
$ sudo docker image rm 1345a7ba5eb6
Error response from daemon: conflict: unable to delete 1345a7ba5eb6 (must be forced) - image is being used by stopped container 69bbed5c06aa
```

报错原因：要删除的镜像正在被容器使用．

解决方法：

- 执行`docker ps -a`命令查看所有容器（包括未运行的容器），找到是哪个容器正在使用这个要删除的镜像(列出的第二列信息即为镜像名称)．
- 执行`docker rm <容器id>`命令删除指定容器．



1. 2 image has dependent child images

```
Error response from daemon: conflict: unable to delete 4e5021d210f6 (cannot be forced) - image has dependent child images
```

原因：有另外的镜像FROM了待删除的镜像．



