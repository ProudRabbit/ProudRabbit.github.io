---
title: Typora+PicGo-core+SMMS图床踩坑记
date: 2020-03-10 10:40:22
tags: [随笔, 软件使用, Typora, Bug]
---

最近把Typora更新后发现，Typora支持PicGo图床工具了。然后赶紧尝鲜，结果踩了不少坑。经过一番百度后发现博主[^LzSkyline]已经解决了，遂按照他的方法解决，但还是踩了坑。于是写了这篇随笔。**PicGo-Core已经解决SMMS V1 API不能上传的问题，直接点击下载或更新，更新下PicGo-Core就行了。**关于PicGo-Core的配置可以看官方的指南。[PicGo-Core中文配置指南](https://picgo.github.io/PicGo-Core-Doc/zh/guide/config.html)

---
由于PicGo-Core已经支持SMMS V2 API了，以下使用插件的方式不用看了，直接将PicGo-Core更新到最新版本即可，然后配置文件参考[PicGo-Core中文配置指南](https://picgo.github.io/PicGo-Core-Doc/zh/guide/config.html)设置即可，懒得去翻阅的小伙伴们，复制下面的配置也行。
```
{
  "picBed": {
    "uploader": "smms", 	// 代表当前的默认上传图床为 SM.MS,
    "smms": {
      "token": "XXXXXXXXXXXXXXXXXX" 	// 从https://sm.ms/home/apitoken获取的token
    }
  },
  "picgoPlugins": {} 		// 为插件预留
}
```
<!--more-->

---

# 一、安装PicGo-Core

打开Typora，进入偏好设置，选择图像，上传服务，选择PicGo-Core，然后安装，安装完成后点击验证图片上传选项。

![image-20200310104640653](https://i.loli.net/2020/03/10/bM1r2gzNpZPCILi.png)

# 二、安装smms v2 API插件

**此乃坑一:SMMS V1 API停用**

正如博主**Lzskyline**所说这是个大坑，有多大呢。PicGo-Core目前使用的是SMMS v1 API，但是SMMS已经把V1 API给停了，只能使用V2。（*PicGo-Core作者已经在Github上表示后面会增加v2 API*）经过查询后发现有其他开发者通过第三方插件的方式解决了这个问题, 所以我们需要安装这个v2版本的smms-user插件.
根据验证时的地址找到PicGo-Core的程序目录
![image-20200310105913161](https://i.loli.net/2020/03/10/soWZrdVcDKNCft1.png)
**因为API不可用，这里肯定会失败，不用管他**

然后我们进入pico的根目录，在命令行中输入

```C
.\picgo.exe install smms-user
```
等待安装完成即可。



----



**坑二：Windows下使用Git Bash不能安装，需要使用 CMD**

打开CMD，输入

```c
CD C:\Users\你自己的用户名\AppData\Roaming\Typora\picgo\win64
```

进入picgo的根目录，然后再输入

```C
.\picgo.exe install smms-user
```



# 三、配置PicGo-Core

安装完成后，点击第一张图中的打开配置文件，替换为博主**Lzskyline**给的代码，

```c
{
  "picBed": {
    "current": "smms-user",
    "uploader": "smms-user",
    "smms-user": {
      "Authorization": "这里替换成你自己的SMMS token"
    },
    "transformer": "path"
  },
  "picgoPlugins": {
    "picgo-plugin-smms-user": true
  }
}
```

没有Authorization的自己去这里申请一个: [SMMS](https://sm.ms/home/apitoken)

# 四、最终效果

我们在图片上右击，上传图片，上传完成后，图片的url会自动更换，很是方便。妈妈再也不用担心我不会插图了。**也可以在Typora中配置插入图片自动上传。**
![image-20200310111959215](https://i.loli.net/2020/03/10/vIBqtJfCD5kyrGd.png)

# 五、总结

虽然踩坑不少, 但毕竟这个功能也是新出的, 相信之后会越来越完善，使用越来越方便。

最后，感谢**LzSkyline**博主提供的教程。



[^LzSkyline]: 该博主地址 https://www.lzskyline.com/archives/87/