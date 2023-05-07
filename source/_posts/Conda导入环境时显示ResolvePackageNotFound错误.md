---
title: Conda导入环境时显示ResolvePackageNotFound错误
date: 2021-09-06 21:14:21
tags: [随笔, 软件使用, anaconda]
keywords: [anaconda, anaconda导入环境, ResolvePackageNotFound]
description: 使用正则去除anaconda导出环境配置时的详细信息，防止导入环境时出现ResolvePackageNotFound错误。
---

﻿﻿﻿`Conda`导入环境时有时候会出现ResolvePackageNotFound错误，该错误的解决方式，博主[哦啦哦啦](https://blog.csdn.net/weixin_42240667/article/details/115773535)已经说明了怎么解决了，但是手动去删除包后面的详细配置信息还是比较麻烦的，我这里给出使用正则去删除的方法。首先用支持正则匹配的文本编辑器打开，推荐`VScode`、`Notepad3`。PS：不推荐`notepad++`，原因的话百度notepad++作者即可知道。

<!--more-->

1. 使用`VScode`或`Notepad3`打开`conda`导出的`yaml`环境文件，然后打开替换，输入如下正则表达式

```re
(?<=^ {2}-.*=.*=).*
```

Notepad3匹配结果，直接点击全部替换即可。
![Notepad3匹配结果](https://img-blog.csdnimg.cn/16bedbfc0f714bae9f0fae573726fc19.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA6Lev55e055qE5YWU5a2Q,size_20,color_FFFFFF,t_70,g_se,x_16)

2. 使用第一步替换后，每一行的行尾漏了一个等号，再使用如下正则表达式全部替换即可。

```re
=$
```

![替换结果](https://img-blog.csdnimg.cn/b399fd228c2d4658ac97da94b3f65580.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA6Lev55e055qE5YWU5a2Q,size_20,color_FFFFFF,t_70,g_se,x_16)
