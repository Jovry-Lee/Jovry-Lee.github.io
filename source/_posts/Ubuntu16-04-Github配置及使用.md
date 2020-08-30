---
title: Ubuntu16.04-Github配置及使用
date: 2020-08-18 17:50:49
tags: ["Ubuntu", "Config", "Git"]
categories: ["Ubuntu", "Config"]
---

#### 一、Git安装

安装命令：
```
sudo apt-get install git
```

<!--more-->

#### 二、Github账号

##### 2.1 注册GitHub账号
注册地址：[官网](https://github.com/)

##### 2.2 SSH配置
配置方式参考：Ubuntu 16.04-设置SSH密钥.md

##### 2.3 GitHub配置
- 去登录github账户，在右上角点击头像找到Settings
- 点进去后点击左侧栏中的SSH and GPG keys
- 点击右侧New SSH key
- 随意输入个能带便当前机器的名字，并把本地.ssh目录下生成的关于github的.pub文件拷贝进来
- 保存即可

##### 2.4 连接测试
终端输入以下命令：
```
$ ssh -T git@github.com
Hi Jovry-Lee! You've successfully authenticated, but GitHub does not provide shell access.
```

表示github配置完成。

#### 三、配置本地Git个人信息
    Git会依据本地设定的用户名和邮箱向远程主机提交更改，Github也是依据这些信息进行权限管理的。
##### 3.1 配置全局个人信息
```
$ git config --global user.name "your github name"
$ git config --global user.email "youraddress@youremail.com"
```

##### 3.2 配置单个项目个人信息
    设置单个项目信息
```
$ git config user.name "your github name"
$ git config user.email "youraddress@youremail.com"
```

#### 四、上传本地项目到GitHub
##### 4.1 创建远程代码仓库
- 点击右上角“＋”旁边的小三角，选择“New repository”；
- 填写代码仓库信息；
- 点击“Create repository”。

##### 4.2 设置本地仓库
```
#把这个目录变成Git可以管理的仓库
git init
# 文件添加到仓库
git add README.md
# 添加所有文件.
git add .
# 提交文件到仓库
git commit -m "first commit"
# 管理远程仓库
git remote add origin <仓库地址>
# 将本地库的所有内容推送到远程库上
git push -u origin master
```
---
#### 参考文档
1. [Ubuntu 16.04下Github配置](https://lrscy.github.io/2017/05/01/Ubuntu-Github-config/)
2. [github入门到上传本地项目](https://www.cnblogs.com/specter45/p/github.html)