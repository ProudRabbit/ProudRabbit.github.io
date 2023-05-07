---
title: Git学习（一）
date: 2020-03-08 14:53:25
tags: [软件使用, Git]
---

# 基础篇

这已经是重写第二遍了，原因没做好Git的项目跟踪，然后VScode误操作，删了。。。。😭😭，还是typora好用，不敢用VScode了。。。。

## 需要工具

	* Git安装包：地址：https://git-scm.com/
	* Github网站账号 地址：https://github.com/

---

<!--more-->

## 一、安装Git

​	百度搜索Git下载安装直接默认即可。然后进入Github注册账号。

​	Git相关知识主要分为三个区，工作区，暂存区以及Git仓库

## 二、进行全局配置

​	右键在空白处右键，点击”Git bash here“打开Git命令行窗口

```c
git config --global user.name“用户名”//绑定用户，使用GitHub注册的用户名。

git config --global user.email“ 邮箱” //绑定邮箱，同样使用GitHub注册用的邮箱。
```

注意不要输错

## 三、创建本地仓库

==不建议在现有项目上学习Git，防止出现无法恢复的误操作。不要使用中文路径名。==

1. `mkdir study_git`    //创建文件夹（项目）
2. `cd study_git`          //进入项目目录
3. Git仓库初始化（让Git知道他需要管理这个项目目录 `git init` 文件夹会添加一个隐藏的文件夹，不能删除也不能随意更改其中的内容。

4. 仓库常用操作指令

1. 1. 查看当前状态： `git status`
   2. 添加到缓存区： `git add 文件名`   //说明git      add指令可以添加一个文件，也可以添加多个文件。

| 语法    | 命令格式                                         |
| ------ | ----------------------------------------------- |
| 语法1： | git  add 文件名                                  |
| 语法2： | git  add 文件名1 文件名2 文件名3……                 |
| 语法3： | git  add .    //【添加当前目录下所有文件到缓存区】    |

6. 提交至版本库： `git commit -m “注释内容”`

后续进行修改文件操作后，重复使用`git add`和`git commit`指令即可。

![clip_image001](https://i.loli.net/2020/03/08/9VXrUPDZAwSCjYm.png)

## 四、版本回退

版本回退分为两步进行操作

查看版本，确定需要回到的时刻点；指令：`git log`  和`git log --pretty = oneline`   //推荐使用第二个

1. 回退操作，指令：`git reset --hard 版本号`

2. 黄色的字符串为某一个时间点提交的序号（版本号）,（HEAD->master）表示当前所在的位置。回到过去后再想回到最新的版本，则需要指令查看历史操作，以得到最新的commit id  指令 `git reflog`

![clip_image002](https://i.loli.net/2020/03/08/jHOBoGuVXd1wyb9.png)

==<u>小结：要想进行版本回退，需要得到commit id，然后进行回退；要想回到未来，需要使用`git reflog`进行历史查看，得到最新的commit id；版本回退时commit id可以不用写全，git自动识别，至少写四位。</u>==

## 六、远程仓库

推荐使用GitHub提供的服务,后面会使用Git同步推送到Github和Gitee(国内码云)，推荐使用SSH公私钥对的方式

### 1. 基于HTTP协议

​		1. 创建空目录，名称为GitHub上创建的仓库名。![clip_image003](https://i.loli.net/2020/03/08/ZN2lCGmeq9Kkdah.png)

​		2. 使用 clone 指令克隆线上仓库到本地。指令：`git clone 线上仓库地址`

​			![clip_image004](https://i.loli.net/2020/03/08/LiUvyBMHqmIoEPf.png)

​		3. 在仓库上做对应操作（提交暂存区、提交本地仓库、提交线上仓库、拉取线上仓库）

​		4. 提交线上仓库的指令：`git push`         //如果第一次提交显示错误403表示没有权限，需要修改“.git/config”文件内容

​			将

​			`[remote"origin"]url = https://github.com/用户名/仓库名.git`

​			修改为

​			`[remote"origin"]url=https://用户名:密码@github.com/用户名/仓库名.git`

​			提交成功后，可以查看线上仓库的内容，需要刷新。

![clip_image005](https://i.loli.net/2020/03/08/Ml7yrNHXZR4cWQJ.png)

 

​			5 拉取线上仓库指令：`git pull`

​				首先在线上创建文件

![clip_image006](https://i.loli.net/2020/03/08/2ZbHRjinu7wN9Xm.png)

 

![clip_image007](https://i.loli.net/2020/03/08/nXWsqceMBmHw1tR.png)

​				然后输入指令：`git pull`

![clip_image008](https://i.loli.net/2020/03/08/nsDFpY3OHrh1SJc.png)

==注意：每天上班前拉取线上最新版本（`git pull`），下班前推送版本（`git push`）将本地代码提交的线上仓库。==

### 2. 基于SSH协议（推荐）

该方式与前面HTTP方式相比，只是影响GitHub对于用户的身份鉴权方式，对于git的具体操作（如提交本地，添加注释等）没有任何影响。

步骤：

​			①生成客户端公私钥对文件（文件默认在C盘用户Admin的.ssh文件夹内）；

​			②将公钥上传到GitHub

​	2.1 生成公私钥对指令（需先自行安装OpenSSH）：`ssh-keygen -t rsa -C "注册邮箱"`

​	2.2 将公钥文件内容上传GitHub（id_rsa.pub）

​	2.3 验证公钥`ssh -T git@github.com` （@后网址可以修改）

![clip_image009](https://i.loli.net/2020/03/08/YJgw8u9xIWRnaL6.png)

 

![clip_image010](https://i.loli.net/2020/03/08/CygzQBHtISkq5TO.png)

 

​	标题可以随意填写，填写完之后保存即可。

​	2.4 执行后续git操作，操作与之前一样。

​			例如克隆线上仓库到本地

​			`Git clone git@github.com:ProudRabbit/STM32F4_FreeRTOS.git  //使用SSH地址`

![lip_image011](https://i.loli.net/2020/03/08/QPmlKxUAEH5Vb6W.png)

 

==<u>PS:在使用SSH或者HTTPS协议克隆线上仓库到本地后，不要修改线上仓库的地址（如使用SSH改为使用HTTPS），这样会导致本地无法访问线上仓库。</u>==