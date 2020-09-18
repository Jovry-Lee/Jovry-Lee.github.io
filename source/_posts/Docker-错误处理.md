---
title: Docker-错误处理
date: 2020-09-16 17:57:09
tags: ["Docker"]
categories: ["Docker"]
---



记录一些遇见的错误，及解决方法．

<!--more-->

#### 1 删除镜像错误

##### 1.1 image is being used by stopped container <xxx>

`报错信息`：

```bash
$ sudo docker image rm 1345a7ba5eb6
Error response from daemon: conflict: unable to delete 1345a7ba5eb6 (must be forced) - image is being used by stopped container 69bbed5c06aa
```

`报错原因`：要删除的镜像正在被容器使用．



`解决方法`：删除正在使用镜像的容器，再进行镜像的删除．

- 执行`docker ps -a`命令查看所有容器（包括未运行的容器），找到是哪个容器正在使用这个要删除的镜像(列出的第二列信息即为镜像名称)．
- 执行`docker rm <容器id>`命令删除指定容器．



##### 1.2 image is referenced in multiple repositories

`报错信息`：

```bash
$ sudo docker image rm $(sudo docker image ls -q)
Error response from daemon: conflict: unable to delete 4e5021d210f6 (must be forced) - image is referenced in multiple repositories
```



`报错原因`：两个仓库指向了同一个镜像id，如下所示：

```bash
$ sudo docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
ubuntu              18.04               4e5021d210f6        6 months ago        64.2MB
ubuntu              latest              4e5021d210f6        6 months ago        64.2MB
```



`解决方法`：通过指定`仓库名：标签`的方式进行删除．

```bash
$ sudo docker image rm ubuntu:18.04
```



#### 2 拉取镜像错误

##### 2.1 read: connection timed out

`报错信息`：

```bash
$ sudo docker pull ubuntu:18.04
18.04: Pulling from library/ubuntu
5d9821c94847: Downloading  4.173MB/26.7MB
a610eae58dfc: Download complete 
a40e0eb9f140: Download complete 
error pulling image configuration: Get https://production.cloudflare.docker.com/registry-v2/docker/registry/v2/blobs/sha256/c1/c14bccfdea1cc6aee142cd95f4069b6ada6b57e484e7886391816e1eba856950/data?verify=1600417610-ZVyDnSC2hWAtmSPbiNbhg4MHxGM%3D: read tcp 172.20.33.9:58120->104.18.124.25:443: read: connection timed out
```

`报错原因`：docker默认使用国外官方网站镜像，速度比较慢，甚至无法连接状态。



`解决方法`：使用国内镜像源．

- docker国内官方镜像地址：`https://registry.docker-cn.com`（貌似已经不能用了）
- 网易：`https://hub-mirror.c.163.com`



`配置方法`：

- 修改 `/etc/docker/daemon.json`文件（<u>该文件默认不存在，需要手动编写</u>．但是注意不要与传递给命令行调用的选项冲突）

```json /etc/docker/daemon.json
{
"registry-mirrors":["<镜像源地址>"]
}
```

- 重启docker生效：

```
systemctl daemon-reload
systemctl restart docker
```

