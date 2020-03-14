---
title: Git学习（三）
date: 2020-03-09 12:17:37
tags: [学习笔记, Git]
---

# 指令篇

​    以下是我自己整理的常用Git命令。分为本地操作命令，远程仓库相关命令，分支操作命令，保护现场操作命令。

<!--more-->

# 一、本地操作命令

```c
git init 				//把当前的目录变成可以管理的git仓库，生成隐藏.git文件
	
git add XX  			//把xx文件添加到暂存区去。

git commit –m “XX”  	//提交文件 –m 后面的是注释。

git status   			//查看仓库状态

git diff  XX     		//查看XX文件修改了那些内容

git log        			//查看历史记录

git reset --hard HEAD^  //或者 git reset  –hard HEAD~数字 回退到上几个版本，不写数字默认为一。

git reset --hard 版本ID  //版本号为使用git log查询到的黄色字符串

git reflog      		//查看历史记录的版本号id

git checkout -- XX  	//把XX文件在工作区的修改全部撤销。

git rm XX         		//删除XX文件,知识删除工作目录和暂存区的文件，也就是取消跟踪

git rm --f XX			//删除XX文件的跟踪，并且删除本地文件，不写文件名默认删除所有文件
    
git rm --cached XX		//删除XX的跟踪，并保留在本地。--cached指的是暂存区，不写文件名为丢弃所有文件

git mu 旧文件名 新文件名   //重命名文件
    
```



----

# 二、远程仓库相关指令

```c
git remote add origin 线上仓库地址	//关联一个远程库 origin可以修改为github或gitee
    
git push –u origin master 			//把当前master分支推送到远程库(第一次要用-u)

git push origin XX 					//Git会把XX分支推送到远程库对应的远程分支上
    
git clone 线上仓库地址				//从远程库中克隆
    
git remote 							//查看远程库的信息
    
git remote rm origin				//删除已关联的名为origin的远程库
    
git remote –v 						//查看远程库的详细信息
    
git pull							//拉取远程仓库
    
git checkout –b dev  origin/dev		//创建dev分支并切换到dev分支上，同时关联远程Dev分支

```

==<u>PS:当你的小伙伴从远程库clone时，默认情况下，你的小伙伴只能看到本地的master分支,需要使用`git checkout –b dev origin/dev` 指令创建远程origin的Dev分支到本地。</u>==



---

# 三、分支操作命令

```c
git branch name  				//创建分支
    
git checkout –b dev  			//创建dev分支 并切换到dev分支上。git checkout –b dev origin/dev同时关联远程Dev分支
    
git branch  					//查看当前所有的分支
    
git checkout master 			//切换回master分支 使用switch也行
    
git merge dev    				//在当前的分支上合并dev分支
    
git merge --no-ff -m "XX" dev	//不使用Fast forward模式合并，合并后被合并(dev)分支依旧保留
    
git branch –d dev 				//删除dev分支
    
git cherry-pick 版本ID		   //复制一个特定的提交到当前分支（常用来修复BUG）

```

# 四、现场保护相关命令

```c
git stash 				//把当前的工作隐藏起来 等以后恢复现场后继续工作
    
git stash list 			//查看所有被隐藏的文件列表
    
git stash apply 		//恢复被隐藏的文件，但是stash内的内容不删除
    
git stash drop 			//删除stash内的文件
    
git stash pop 			//恢复文件的同时 stash的内容删除

```

