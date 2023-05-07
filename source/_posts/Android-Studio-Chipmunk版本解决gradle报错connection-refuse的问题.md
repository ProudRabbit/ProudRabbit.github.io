---
title: Android Studio Chipmunk版本解决gradle报错connection refuse的问题.md
date: 2022-08-27 10:57:12
tags: [随笔, 安卓, gradle]
keywords: [Android-Studio-Chipmunk, gradle]
description: 解决Android Studio Chipmunk版本在使用gradle解决依赖时报错connection refuse的问题。2022年
---

## Android Studio Chipmunk版本解决gradle报错connection refuse的问题

[toc]

### 一、问题出现的原因

​	因为gradle使用的仓库是国外的，很多时候访问不了，或者公司里面进行了管控导致无法下载依赖，所以gradle会同步出错。目前网上的教程都比较老，已经不适用于Android Studio Chipmunk版本了，因此摸索出了这个方法。

### 二、解决办法

1. 使用KX工具，不会翻墙的程序员不是一个好程序员。

2. 使用国内镜像源

   这里推荐推荐阿里云的镜像源<https://developer.aliyun.com/mvn/guide>，地址是上面的地址。添加方法有三种，分别是工程添加，全局添加，使用工程模板。

   - 工程模板：安卓北极狐版本后添加工程模板较为繁琐，暂不推荐。<https://www.jianshu.com/p/fa9b1357ebe7>

   - 工程添加：

     在工程根目录下的 `settings.gradle`文件下，修改部分内容为以下内容。

     ```groovy
     dependencyResolutionManagement {
         repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
         repositories {
             maven { url 'https://maven.aliyun.com/repository/public'}
             maven { url 'https://maven.aliyun.com/repository/google'}
             maven { url 'https://maven.aliyun.com/repository/gradle-plugin'}
             
             google()
             mavenCentral()
         }
     }
     ```

   - 全局添加：

     在`C:\Users\your_username\.gradle\wrapper\dists\gradle-7.3.3-bin\一串乱码\gradle-7.3.3\init.d` 文件夹下新建一个 `.gradle` 结尾的文件，如 `init.gradle`，然后输入以下内容，这样每次在 gradle 执行的时候会首先加载这个文件设置仓库。

     ```groovy
     settingsEvaluated { settings -> 
         settings.dependencyResolutionManagement {
         	repositories {
             	maven { url 'https://maven.aliyun.com/repository/public'}
             	maven { url 'https://maven.aliyun.com/repository/google'}
             	maven { url 'https://maven.aliyun.com/repository/gradle-plugin'}
                 mavenLocal()
             	mavenCentral()
             	google()
         	}
     	}
     }
     ```

     
