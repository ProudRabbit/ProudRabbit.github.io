---
title: git commit --amend踩坑
date: 2020-03-14 17:16:26
tags: [Git, 随笔, 软件使用, Bug]
---

因为改动比较小，所以我不想重建一个commit，于是我是用了`git commit --amend`命令,由于之前已经将该commit推送到远程仓库，导致修改后推送失败。百度后发现如果你的commit已经push到了远程仓库，那么使用`--amend`修改commit后，`git push`时一定要使用 `--force-with-lease` 参数来强制推送，否则就会报错。

<!--more-->

# 这是我自己推送失败的例子

![image-20200314172358176](https://i.loli.net/2020/03/14/oTx6a7ZyFP23gmE.png)

# 解决方式

## 一、第一种

使用后`git commit --amend -m "修改Git学习(三)指令"`

==**注意：-m “这里的内容和要追加的commit相同即可”，当然你也可以修改**==

推送时使用`git push --force-with-lease gitee Backup`命令。



百度到的教程来自于CSDN博主 [无色云](https://blog.csdn.net/weixin_38669561/article/details/103385514)

## 二、第二种

**这种用处不大，不建议使用，~~因为无异于脱裤子放屁~~。**

根据提示先从远程仓库git pull下来，然后修改冲突的文件后再git add  git commit最后再git push