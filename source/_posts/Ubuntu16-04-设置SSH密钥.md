---
title: Ubuntu16.04-设置SSH密钥
date: 2020-08-19 15:20:20
tags: ["Ubuntu","Config", "SSH"]
categories: ["Ubuntu", "Config"]
---

#### 1 简介
SSH（Secure shell）适用于管理服务器与服务器通信的加密协议。

<!--more-->

#### 2 设置步骤
##### 2.1 创建RSA密钥对
```
ssh-keygen
```
①、默认情况下，ssh-keygen将会创建一个2048位的RSA密钥对，此时已经足够安全，若有特殊需求，可以选择传入-b 4096标志来创建更大的4096位的密钥。

②、执行命令后，终端将会输出一下内容，按ENTER键将密钥保存到.ssh/主目录的子目录中，或指定备用路径。
```
Generating public/private rsa key pair.
Enter file in which to save the key (/home/seven/.ssh/id_rsa):
```
③、若在此之前已经生成了ssh密钥对，则可能看到以下提示,此时若选择覆盖磁盘上的密钥，则无法使用以前的密钥进行身份验证（==这是一个无法逆转的破坏性过程==）
```
/home/seven/.ssh/id_rsa already exists.
Overwrite (y/n)?
```
④、然后会看到以下输出，此处可以选择输入安全密码，建议使用。密码短语增加了额外的安全层，以防未经授权的用户登录。

##### 2.2 将公钥复制到Ubuntu服务器
###### 2.2.1 方法一 使用实用程序ssh-copy-id
使用实用程序ssh-copy-id，只需要指定要连接的远程主机，以及具有ssh访问密码的用户账户即可。
```
ssh-copy-id username@remote_host
```

###### 2.2.2 方法二 使用ssh复制公钥
使用cat命令读取本地计算机上的公钥内容，并通过ssh连接到远程服务器来管理它。（==注：使用cat命令，而不要使用vim去复制，可能会复制出奇怪的字符进去==）
```
cat ~/.ssh/id_rsa.pub | ssh username@remote_host "mkdir -p ~/.ssh && touch ~/.ssh/authorized_keys && chmod -R go= ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

###### 2.2.3 方法三 手动复制公钥
```
cat ~/.ssh/id_rsa.pub
```
复制公钥，再登录服务器，执行以下命令
```
echo public_key_string >> ~/.ssh/authrized_keys
```
其中，public_key_string为复制的公钥

#### 3 使用SSH密钥对Ubuntu服务器进行身份验证
以上配置完成后，则可以在没有远程账户密码的情况下登录远程主机。
登录命令：
```
ssh username@remote_host
```
若是在第一次连接此主机，则可能会出现以下内容,这表示当前本地计算机无法识别远程主机，直接输入YES，然后按ENTER继续。
```
Output
The authenticity of host '203.0.113.1 (203.0.113.1)' can't be established.
ECDSA key fingerprint is fd:fd:d4:f9:77:fe:73:84:e1:55:00:ad:d6:6d:22:fe.
Are you sure you want to continue connecting (yes/no)? yes
```

若步骤一中创建RSA密钥对时没有提供密码，则可以继续登录。若提供了密码，则需要输入密码。

#### 4 在服务器上禁用密码身份验证
详情见参考文献1.


---
#### 参考资料
1. [如何在Ubuntu 16.04上设置SSH密钥](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys-on-ubuntu-1604)

