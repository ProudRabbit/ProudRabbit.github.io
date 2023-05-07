---
title: Git学习（二）
date: 2020-03-09 10:47:48
tags: [软件使用, Git]
---

# 提高篇

​	这是我自己当初学习的笔记，可能不全，这里推荐廖雪峰的的Git教程，很全面。[廖雪峰的Git教程](https://www.liaoxuefeng.com/wiki/896043488029600) 希望对大家有所帮助。

<!--more-->

## 一、分支管理

​		Git每次提交后都有记录，Git把他们串成时间线，形成类似于时间轴的东西，这个时间轴就是一个分支，我们称之为master分支。在开发是往往是团队合作多人同时开发，如下图所示，因此需要多个分支，保证分支之间互不干扰。

![用 户 块  商 品 块  订 单 块  物 流 块 ](https://i.loli.net/2020/03/09/ALwzZ3gTjSD9y8x.png)

1. 分支相关指令：

| 功能 |  命令    |
| ---------- | -----------------------------------------------------|
| 查看分支： | `git branch`    |
| 创建分支： | `git  branch 分支名`                                    |
| 切换分支： | `git  checkout 分支名`                                  |
| 删除分支： | `git  branch -d 分支名`                                 |
| 合并分支： | `git  merge 被合并的分支名`                              |
| 备注：     | 当前分支前有个标记“*”并且会高亮，切换分支指令在切换到不存在分支时会自动创建分支。 |

2. 查看、创建、切换分支操作

![未命名图片](https://i.loli.net/2020/03/09/nyj2gwrGs98bmMK.png)

3. 合并分支

     先在Dev分支下修改readme文件，并提交本地。然后切换到master分支，这时发现刚才在Dev分支下readme.txt修改的内容不存在了。此时需要将Dev分支与master分支的内容合并，此时在master分支下已经可以看到Dev修改的内容了。

![8STujg.png](https://s2.ax1x.com/2020/03/09/8STujg.png)

此时Dev分支已经没有用了，可以选择删除了。

==注意：在删除分支时，一定要先退出要删除的分支，然后才能删除。==

执行`git push`将仓库内容提交线上。

##  二、冲突的产生与解决

 ### 1. 模拟产生冲突

   1. 同事下班后修改了线上仓库代码。

      ![未命名图片3.png](https://i.loli.net/2020/03/09/wargxJ47pVfvkWF.png)

   2. 第二天我没有拉取线上内容，直接修改了本地仓库的内容，然后提交了线上仓库。此时提示错误，并且线上仓库并没有改变。

      ![8STDER.png](https://s2.ax1x.com/2020/03/09/8STDER.png)

### 2.解决冲突


   1. git提示我们需要在在此push前先执行`git pull`操作。

   2. `git pull`  提示我们已经将本地与线上仓库的冲突合并到了readme.txt文件

        ![未命名图片5](https://i.loli.net/2020/03/09/crT8Ko2Vu5OvzgP.png)

   3. 打开冲突文件，解决冲突

       ![未命名图片6](https://i.loli.net/2020/03/09/kYmhcUqNFJ12DZI.png)

  ==解决方法：需要和同时（谁之前提交的）进行商量，看代码如何保留，将改好的代码重新提交即可。==

   4. 冲突标记说明
      从`<<<<<<<`和`======`这里开始的行之间的一行（或多行）就是你在本地已经拥有的东西 ，因为`HEAD`指向你当前的分支或提交。`=======`和`>>>>>>>`是另一个（拉）提交引入的内容，在本例中是`b2121b` 。 这是合并到`HEAD`中的提交的对象名称。


## 三、图形管理工具

   1. GitHub for Desktop

      老牌的Git GUI管理工具，功能丰富，基本操作和高级操作都非常流畅，适合初学者。

   2. TortoiseGit

      简称tgit，中文海龟Git，适合熟悉SVN的开发人员.

## 四、忽略文件

​     项目中可能会有很多万年不变的文件目录，例如css、images等，或者有修改但是不想提交到远程仓库的文档，此时我们可以使用“忽略文件”机制来实现。

​	忽略文件需要新建一个名为.gitignore的文件，该文件用于申明忽略文件或不忽略文件的规则，规则对当前目录及其子目录生效。

==注意：该文件因为没有文件名，没发直接在windows目录下直接创建，可是通过命令行git bash来touch创建。==

   1. 常见规则写法
| 命令 |  功能  |
| :---------- | :-----------------------------------------------------|
| mtk/               | 过滤整个文件夹                              |
| *.zip              | 过滤所有.zip文件                            |
| mtk/文件名.后缀名  | 过滤某个具体文件                            |
| !文件名.后缀名     | 不过滤具体某个文件，!表示不过滤某个具体文件 |
| !mtk/文件名.后缀名 | 不过滤某个文件夹下的具体文件                |
| 备注               | 在文件以#开头的都是注释                     |

==注意：.gitignore只能忽略那些原来没有被track的文件，如果某些文件已经被纳入了版本管理中，则修改.gitignore是无效的。==

2. 先在本地仓库中新建一个js目录以及目录中js文件
3. 依次提交本地与线上
4. 新增.gitignore文件 指令：`touch .gitignore`

![未命名图片7](https://i.loli.net/2020/03/09/ewLz8Ygs19rT2Wc.png)

5. 编写规则（根据需要编写）

![未命名图片8](https://i.loli.net/2020/03/09/Yk9hFtnxy7Le5Rm.png)

6. 添加pass.txt文件后再次提交线上仓库，发现线上并没有pass.txt文件，说明规则已经生效。

![未命名图片9](https://i.loli.net/2020/03/09/s1WBJdehCIo7q5K.png)

![未命名图片10](https://i.loli.net/2020/03/09/qOacJKrxun8bUz2.png)

![未命名图片11](https://i.loli.net/2020/03/09/PBVzGkwEDdLZrRN.png)

## 五、同时提交到Github和Gitee仓库
1. 注册Gitee账号，**<u>==注意邮箱要和Github注册所使用的邮箱一致。==</u>**

2. 添加SSH公钥到Gitee，ssh在 [Git学习（一）六、远程仓库](./Git学习（一）.html#2-基于SSH协议（推荐）) 六、远程仓库，基于SSH篇章提到怎么使用。

   ![捕获](https://i.loli.net/2020/03/09/8PmMBHTJNc9CioU.jpg)

3. 输入`ssh -T git@gitee.com`验证。第一次验证会让你确实是否是你本人操作，输入yes即可。

   ![捕获1](https://i.loli.net/2020/03/09/Y7HDpK2GPuThJie.jpg)

4. 输入`git remote rm origin`删除已关联的名为origin的远程库。
5. 输入`git remote add github 线上仓库地址`关联github上的远程仓库
6. 输入`git remote add gitee 线上仓库地址` 关联gitee上的远程仓库
7. 输入`git remoter -v`查看本地仓库关联的远程仓库

![捕获2](https://i.loli.net/2020/03/09/m8VlEor3wWeD7qI.jpg)

8. 输入`git push github 分支名`和`git push gitee 分支名`分别推送到对应仓库上的对应分支上。至此完成了本地仓库同步推送到Github和Gitee上的远程仓库。

