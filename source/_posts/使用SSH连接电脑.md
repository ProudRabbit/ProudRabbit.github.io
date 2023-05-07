---
title: 使用SSH连接电脑
date: 2020-07-02 12:10:02
tags: [随笔, 软件使用, SSH]
keywords:
description:
---

# 如何使用SSH来远程连接电脑  
本篇文章主要是描述如何使用`SSH`来远程连接`Linux`主机（Ubuntu）的用户，也适用于其他的Linux发行版。

<!--more-->

## 1. 客户端安装SSH

### 1.1 Ubuntu

```c++
sudo apt install SSH
```

### 1.2 Windows

Windows 10 1809默认安装了`OpenSSH`，无需安装。  

## 2. 服务器安装SSH-server  

由于安装方式和第一步一样，这里就之列出`Ubuntu`下的安装方式。  
```c
sudo apt install SSH-server
```
**PS：到这里已经可以连接了，下面是使用SSH免密登录，可以使用  `Ssh 用户名@服务器地址`  来连接服务器**  

## 3. 客户端生成公私钥  

```c
ssh-keygen -t rsa
```
文件位置在用户家目录下，如`Ubuntu`下就在`~/.ssh`下，由于是隐藏文件，请打开显示隐藏文件查看。  

## 4. 上传公钥到服务器  

```c
ssh-copy-id -p 22 用户名@服务器IP地址
```
提示授权时 输入yes回车，然后提示输入服务器用户的密码。  手动复制到服务器上也行。

----

手动复制如下，
> 将客户端的`.ssh`文件夹下的`id_rsa.pub`文件内的内容复制
> 粘贴到服务器端的`.ssh`文件夹下的`authorized_keys`文件内。
> 如果服务器端`authorized_keys`文件不存在，请自行创建。

## 5. 连接服务器

```c
ssh 用户名@服务器地址
```

## 6. 给服务器取别名，免除每次要输入地址  
```c
touch ~/.ssh/config  

vim ~/.ssh/config
```
然后在文件里输入
```c
Host 别名
	HostName 服务器IP地址
	User 你要连接的服务器上的用户名
	Port 22
```
然后客户端使用`ssh 别名`即可连接服务器。  
其实就是使用别名来代替用户名@IP这一串字符，22是SSH默认使用的端口号，不建议修改。  

## 7. 文件传输

1. 传文件的话，输入
```c
 scp 文件 用户名@域名/ip:服务器上的路径
```
 如果使用config文件配置过名称后，可以使用
```
scp 文件 别名:服务器上的路径
```

2. 同理，传送文件夹  
```c
scp -r 文件夹 用户名@域名/ip:服务器上的路径
```

3. 下载远程文件  
```c
 scp 用户名@域名/ip:远程文件的路径 本地路径
```

4. 下载远程文件夹  
```c
 scp -r 用户名@域名/ip:远程文件夹的路径 本地路径
```

------
**PS：如果需要连接`root`账号，需要修改服务器`/etc/ssh/sshd_config文件`,然后输入`service ssh restart`重启SSH服务。**

```
#PermitRootLogin no   改为  PermitRootLogin yes；
```



----



## 8. 可能会用到的一些SSH命令

```c
//验证命令
ssh -T 用户名@域名/ip
```

