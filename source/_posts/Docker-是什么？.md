---
title: Docker-是什么？
date: 2020-09-16 11:54:52
tags: ["Docker"]
categories: ["Docker"]
---



#### 1 Docker是什么

Docker是使用Google推出的Go语言开发的，基于Linux内核的cgroup, namespace,OverlayFS类的Union FS等技术，对进程进行封装隔离的操作系统层面的虚拟化技术．

<!--more-->

由于隔离的进程独立于宿主和其他的隔离程序的进程，因此也称为`容器`．



##### 1.1 Docker与传统虚拟化方式的不同点

`Docker作为一种新兴的虚拟化技术，与传统的虚拟机技术相比有那些不同点呢？`

- `传统虚拟机技术`是虚拟出一套硬件后，在其上运行一个完整的操作系统，在该系统上在运行所需应用进程。
- `Docker`的应用进程直接运行在宿主的内核，容器内没有自己的内核，而且没有进行硬件虚拟，相比传统虚拟机更加轻便。



##### 1.2 Docker相比虚拟机的优势

- `更高效的利用系统资源`：容器不需要进行硬件虚拟及运行完整操作系统等额外开销，Docker对系统资源的利用率更高。
- `更快速的启动时间`：虚拟机启动应用服务往往需要数分钟；Docker容器应用，直接运行于宿主内核，无需启动完整的操作系统，启动时间可以做到秒或毫秒级。
- `一致的运行环境`：Docker的镜像提供除了内核外完整的运行时环境，确保了应用运行环境一致性。
- `持续交付和部署`：使用Docker可以通过定制应用镜像来实现持续集成、持续交付、部署。开发人员可以通过Dockerfile来进行镜像构建，并结合持续集成系统进行集成测试；运维人员则可以直接在生产环境中快速部署该镜像，甚至结合持续部署系统进行自动部署。
- `更轻松的迁移`：Docker确保了执行环境的一致性，使得应用的迁移更加容易。
- `更轻松的维护扩展`：Docker使用的分层存储以及镜像的技术，使得应用重复部分的复用更加容易，也使得应用的维护更新更加简单，基于基础镜像进一步扩展镜像也变得非常简单.



#### ２ Docker的三大核心概念

- `镜像（Image）`：Docker镜像是一个reshuffle的文件系统，除了提供容器运行时所需的程序、库、资源、配置等文件外，还包括了一些为运行时准备的一些配置参数（如：匿名卷、环境变量、用户等）。镜像不包含任何动态数据，其内容在构建之后也不会被改变。
- `容器（Container）`：容器的是指是进程，但与直接在宿主机执行的进程不同，容器进程运行于属于自己的独立的命名空间。容器拥有自己的root文件系统、自己的网络配置、自己的进程空间，甚至自己的用户ID空间。容器内的进程是运行在一个隔离的环境里，使用起来，就好像是在一个独立于宿主的系统下操作一样。这种特性使得容器封装的应用比直接在宿主机运行更加安全。
- `仓库（Repository）`：集中的存储、分发镜像的地方。





------

#### 参考资料：

 [Docker —— 从入门到实践](https://yeasy.gitbook.io/docker_practice/)

[十分钟看懂Docker和K8S](https://zhuanlan.zhihu.com/p/185752438)