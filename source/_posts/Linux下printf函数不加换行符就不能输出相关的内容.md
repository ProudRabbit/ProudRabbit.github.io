---
title: Linux下printf函数不加\n就不能输出相关的内容
date: 2021-01-11 14:30:09
tags: [随笔, Linux]
keywords:
description: Linux下printf函数不加\n就不能输出相关的内容
---

**转载自：**[CSDN博主Alen.Wang](https://blog.csdn.net/qq_26093511/article/details/53255970) 地址：<https://blog.csdn.net/qq_26093511/article/details/53255970>

**原因：  输出缓冲区的问题。**

`unix`上标准输入输出都是带有缓存的，一般是行缓存。

对于标准输出，需要输出的数据并不是直接输出到终端上，而是首先缓存到某个地方，当遇到行刷新标志或者该缓存已满的情况下，才会把缓存的数据显示到终端设备上。

`ANSI C`中定义换行符`\n`可以认为是行刷新标志。所以，`printf`函数没有带`\n`是不会自动刷新输出流，直至缓存被填满。

<!--more-->



解决方案：

方案1、在`printf`里加`\n`

方案2、在`printf`后面调用`fflush(stdout)`函数来刷新标准输出缓冲区，把输出缓冲区里的东西打印到标准输出设备上 。

方案3、使用`setvbuf(stdout,NULL,_IONBF,0);`函数来禁止缓冲区，这样就会直接进行输出。




这两个函数都是有关流缓冲区的. 具体使用和说明网上有很多.  我只说一下什么是流缓冲区, 是做什么用的。

操作系统为减少IO操作所以设置了缓冲区。等缓冲区满了再去操作IO，这样是为了提高效率。



下面是测试代码：

方案一、

```c
#include<stdio.h>
#include<unistd.h>
 
void main()
{
    int i;
    for(i=0;i<10;i++)
    {
        printf("\r %d%% is complete.\n",i);
        sleep(1);
    }
    printf("\n");
}
```

方案二、

```c
#include<stdio.h>
#include<unistd.h>
 
void main()
{
    int i;
    for(i=0;i<10;i++)
    {
        printf("\r %d%% is complete.",i);
        fflush(stdout);
        sleep(1);
    }
    printf("\n");
}
```

方案三、

```c
#include<stdio.h>
#include<unistd.h>
 
void main()
{
    int i;
    setvbuf(stdout,NULL,_IONBF,0); //直接将缓冲区禁止了. 它就直接输出了
    for(i=0;i<10;i++)
    {
        printf("\r %d%% is complete.",i);
        sleep(1);
    }
    printf("\n");
}
```

