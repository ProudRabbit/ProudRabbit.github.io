---
title: 使用代理解决ros安装过程中rosdep update连接超时的问题
date: 2021-03-05 18:31:32
tags: [Linux, ros]
keywords: rosdep update超时 ros
description: 使用代理解决ros安装过程中rosdep update连接超时的问题
---

# 使用代理解决ROS安装过程中rosdep update连接超时的问题

相信好多朋友都会遇到在使用`sudo rosdep init`和`rosdep update`时连接超时的问题。网上的教程都是改`/etc/hosts`文件来解决的，如果修改了`host`解决了`rosdep update`连接超时的问题，那下面的教程就不需要看了。假如你修改`host`后问题依旧，那么恭喜你，你和我的网络一样差。（总算有人和我的网络一样垃圾了\^__\^）希望下面的教程对你有点帮助。

**本人系统是Ubuntu 18.04.5**

<!--more-->

## 一、下载Shadowsocks-Qt5

首先需要下载`Shadowsocks-Qt5`软件，

```shell
wget https://github.com/shadowsocks/shadowsocks-qt5/releases/download/v3.0.1/Shadowsocks-Qt5-3.0.1-x86_64.AppImage
```

给该文件添加可执行权限，然后执行

```shell
chmod a+x Shadowsocks-Qt5-3.0.1-x86_64.AppImage

sudo ./Shadowsocks-Qt5-3.0.1-x86_64.AppImage
```

这时会打开软件，一开始是什么都没有的，然后点击`连接->添加->手动` ，根据你自己的服务商来填写里面的参数。(PS：感谢师弟！) 最后点击保存，然后链接。

![1](https://i.loli.net/2021/03/05/s9KejYDxBJ6Q2d4.png)![2](https://i.loli.net/2021/03/05/uObmrsHyfeV8BLx.png)

## 二、修改Ubuntu的设置中的网络代理

打开Ubuntu的`设置->网络->网络代理`点击后面的齿轮，选择手动填写`Socks主机`中的参数，地址就是上面添加`Shadowsocks-Qt5`的连接中显示的地址和端口。这个时候打开浏览器就可以进谷歌了，然后你会发现，只有浏览器走代理，终端还是依旧不走啊QAQ。别急，接下来才是重头戏。

![3](https://i.loli.net/2021/03/05/UBXrVcRaFKWzQM4.png)

## 三、使终端走代理

首先需要安装终端代理神器 `proxychains`

```shell
sudo apt install proxychains4
```

在使用`apt`安装时会有`proxychains`和`proxychains4`两个版本，我用的是`proxychains4`。

安装好需要修改配置文件

```shell
sudo vim /etc/proxychains4.conf
```

在最后一行将

```
[ProxyList]
socks4 127.0.0.1 9050
```

修改为还是上面`Shadowsocks-Qt5`的连接中显示的地址和端口

```
[ProxyList]
socks5 127.0.0.1 1080
```

因为我使用的是本地 socks5 的代理，其他的可以根据他给的例子填写。写好后记得`wq`保存推出。

到此就配置好了，要让终端走代理只需要在命令前面添加`proxychains4`即可。例如

```shell
proxychains4 rosdep update
```

接下来就是见证奇迹的时刻。

![4](https://i.loli.net/2021/03/05/McdtJk4wLE8ojWG.png)

**不用的时候记得把`Shadowsocks-Qt5`关闭和Ubuntu中的网络代理设置改为禁止，不然会没有网络。**

## 感谢下面博主所提供的教程

<https://my.oschina.net/chinaliuhan/blog/3065303>

<https://www.cnblogs.com/wAther/p/10472889.html>

<https://blog.csdn.net/u011745228/article/details/103588004?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-4.control&dist_request_id=&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-4.control>