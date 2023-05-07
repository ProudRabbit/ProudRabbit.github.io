---
title: Git学习（三）
date: 2020-03-09 12:17:37
tags: [软件使用, Git]
---

# 指令篇

​    以下是我自己整理的常用Git命令。分为本地操作命令，远程仓库相关命令，分支操作命令，保护现场操作命令。

<!--more-->

# 一、本地操作命令

```shell
git init 						//把当前的目录变成可以管理的git仓库，生成隐藏.git文件
	
git add XX  					//把xx文件添加到暂存区去。

git commit –m “XX”  			//提交文件 –m 后面的是注释。
    
git commit --amend -m "XX"		//更正最近的一次提交
    
git status   					//查看仓库状态

git diff  XX     				//查看XX文件修改了那些内容

git diff > commit.patch			//生成补丁文件，还未提交的修改，不包含commit信息

git diff --cached > commit.patch	//生成补丁文件，已经add但是未commit的修改。或 git diff --staged

git diff HEAD 					//查看已缓存的与未缓存的所有改动

git apply --stat commit.patch	//检查patch文件

git apply --check commit.patch	//检查patch是否可以应用

git apply commit.patch			//打补丁

git apply --reject commit.patch	//自动合入 patch 中不冲突的代码，同时保留冲突的部分
								//同时会生成后缀为 .rej 的文件，保存没有合并进去的部分的内容，可以参考这个进行冲突解决。
								
git format-patch 				//生成补丁文件，包含commit信息

git am commit.patch				//打补丁，针对 git format-patch 生成的补丁：

git log        					//查看历史记录

git log --stat					//git查看历史提交修改了哪些文件

git log --stat -<number> 		//限制显示历史提交的数量

git reflog      				//查看历史记录的版本号id

git reset --hard HEAD^  		//撤销之前的commit，并且舍弃之前commit的修改，或者 git reset  –hard HEAD~数字 回退到上几个版本，不写数字默认为一。

git reset --soft HEAD^ 			//撤销之前的commit，并且保留之前的commit修改

git reset --hard 版本ID		   	//版本号为使用git log查询到的黄色字符串

git reset HEAD <file>			//把暂存区的修改撤销掉（unstage），重新放回工作区

git checkout -- XX  			//把XX文件在工作区的修改全部撤销。

git clean -nxfd					//删除未跟踪的文件，-n显示将要被删除的文件
								//-f删除 untracked files  -fd untracked的目录也一起删掉  						
								//-xfd不管他是否是.gitignore文件里面指定的文件夹和文件,都会删除

git rm XX         				//删除XX文件,知识删除工作目录和暂存区的文件，也就是取消跟踪

git rm --f XX					//删除XX文件的跟踪，并且删除本地文件，不写文件名默认删除所有文件
    
git rm --cached XX				//删除XX的跟踪，并保留在本地。--cached指的是暂存区，不写文件名为丢弃所有文件

git mv 旧文件名 新文件名   		//重命名文件

git tag <tagname> [commit id]	//用于新建一个标签，默认为HEAD，也可以指定一个commit id

git tag -a <tagname> -m "标签信息"   //可以指定标签信息

git show <tagname>					//查看标签信息

git tag								//可以查看所有标签

git tag -d <tagname>				//可以删除一个本地标签；创建的标签都只存储在本地，不会自动推送到远程。所以，打错的标签可以在本地安全删除。
```



----

# 二、远程仓库相关指令

```shell
git remote 							//查看远程库的信息
    
git remote rm origin				//删除已关联的名为origin的远程库
    
git remote –v 						//查看远程库的详细信息

git remote add origin 线上仓库地址	//关联一个远程库 origin可以修改为github或gitee

git clone 线上仓库地址				//从远程库中克隆 

git clone 线上仓库地址 --recursive    //递归克隆整个项目 

git push							//将本地当前分支 推送到 与本地当前分支同名的远程分支上
    
git push origin XX 					//将本地当前分支 推送到 与本地当前分支同名的远程分支上

git push origin 本地分支名:远程分支名	//将本地当前分支 推送到 远程指定分支上

git pull							//将与本地当前分支同名的远程分支 拉取到 本地当前分支上

git pull origin	远程分支名			//将远程指定分支 拉取到 本地当前分支上

git pull origin 远程分支名:本地分支名	//将远程指定分支 拉取到 本地指定分支上
    
git checkout –b dev  origin/dev		//创建dev分支并切换到dev分支上，同时关联远程Dev分支 克隆线上仓库后使用

git push --set-upstream origin 本地分支名 	//将本地分支与远程同名分支相关联
// 简写方式
git push -u origin 本地分支名

git push origin <tagname>			// 可以推送一个本地标签；

git push origin --tags				//可以推送全部未推送过的本地标签；

git push origin :refs/tags/<tagname>	  	//可以删除一个远程标签。如果标签已经推送到远程，要删除远程标签就麻烦一点，先从本地删除 `git tag -d <tagname>` 然后使用该命令从远程删除
```

==<u>PS:当你的小伙伴从远程库clone时，默认情况下，你的小伙伴只能看到本地的master分支,需要使用`git checkout –b dev origin/dev` 指令创建远程origin的Dev分支到本地。</u>==



---

# 三、分支操作命令

```shell
git branch name  				//创建分支
    
git checkout –b dev  			//创建dev分支 并切换到dev分支上。git checkout –b dev origin/dev同时关联远程Dev分支
    
git branch  					//查看当前所有的分支
    
git checkout master 			//切换回master分支 使用switch也行
    
git merge dev    				//在当前的分支上合并dev分支
    
git merge --no-ff -m "XX" dev	//不使用Fast forward模式合并，合并后被合并(dev)分支依旧保留
    
git branch –d dev 				//删除dev分支
    
git cherry-pick 版本ID		   //复制一个特定的提交到当前分支（常用来修复BUG）
```

# 四、现场保护相关命令

```shell
git stash 				//把当前的工作隐藏起来 等以后恢复现场后继续工作
    
git stash list 			//查看所有被隐藏的文件列表
    
git stash apply 		//恢复被隐藏的文件，但是stash内的内容不删除
    
git stash drop 			//删除stash内的文件
    
git stash pop 			//恢复文件的同时 stash的内容删除
```

# 五、子模块使用

```shell
git submodule add <url> <path>	//仓库中添加子模块 url为子模块的路径，path为该子模块存储的目录。

// 克隆带有子模块的仓库后，子模块目录默认无任何内容，需要使用一下命令同步子模块
git submodule init && git submodule update
// 等同于
git submodule update --init --recursive

git submodule update			//更新子模块
    
#******************** 删除子模块 *******************
# 删除子模块文件夹
git rm --cached test
rm -rf test
#删除.gitmodules文件夹中相关子模块信息 
[submodule "assets"]
  path = test
  url = https://github.com/cain/cain-test.git
#删除.git/config文件夹中的相关子模块信息
[submodule "test"]
  url = https://github.com/cain/cain-test.git
#删除.git文件夹中的相关子模块文件
rm -rf .git/modules/test
```

