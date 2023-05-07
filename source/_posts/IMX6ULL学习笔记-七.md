---
title: IMX6ULL学习笔记(七)
date: 2020-09-18 23:14:24
tags: [嵌入式, IMX6ULL, 学习笔记]
keywords:
description: 正点原子alpha开发板IMX6ULL裸机开发学习笔记。
---

**IMX6ULL裸机开发学习**

以下内容是我在学习正点原子`IMX6ULL`开发板`alpha`中记录的笔记，部分摘录自正点原子`IMX6ULL开发手册`。

## 1. 根文件系统简介

根文件系统就是一个特殊的”文件夹“，这个特殊的“文件夹”中保存着Linux运行所必须的，但是无法放入内核里面去。比如命令、库、配置文件等等。

<!--more-->

## 2. 构建根文件系统

初学使用busybox来构建，做项目时使用成熟化的根文件系统构建方式，buildroot，yocto。

1. 修改Makefile，添加交叉编译器

   ```makefile
   ARCH ?= arm
   
   CROSS_COMPILE ?= /usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-g     nueabihf/bin/arm-linux-gnueabihf-
   ```

2. 修改busybox，添加中文字符支持 

   修改`libbb/printable_string.c`中的`printable_string`函数   

   ```c
   	if (c < ' ')
       	break;
   /* 		if (c >= 0x7f)
   			break; */
   	s++;
   ```

   ```c
   if (c == '\0')
       break;
   if (c < ' ')
       *d = '?';
   d++;
   ```

   修改`libbb/unicode.c`中的`unicode_conv_to_printable2`函数内容  

   ```c
   if (unicode_status != UNICODE_ON) {
       char *d;
       if (flags & UNI_FLAG_PAD) {
           d = dst = xmalloc(width + 1);
           while ((int)--width >= 0) {
               unsigned char c = *src;
               if (c == '\0') {
                   do
                       *d++ = ' ';
                   while ((int)--width >= 0);
                   break;
               }
               /* *d++ = (c >= ' ' && c < 0x7f) ? c : '?'; */
               *d++ = (c >= ' ') ? c : '?';
               src++;
           }
           *d = '\0';
       } else {
           d = dst = xstrndup(src, width);
           while (*d) {
               unsigned char c = *d;
               /* if (c < ' ' || c >= 0x7f) */
               if (c < ' ')
                   *d = '?';
               d++;
           }
       }
       if (stats) {
           stats->byte_count = (d - dst);
           stats->unicode_count = (d - dst);
           stats->unicode_width = (d - dst);
       }
       return dst;
   }
   ```

3. 配置busybox

   ```makefile
   make defconfig
   ```

   打开图形化界面，进行相关配置。

   ```makefile
   make menuconfig
   ```

   然后编译busybox

   ```makefile
   make install CONFIG_PREFIX=/home/rabbit/linux/nfs/rootfs
   ```

4. 向根文件系统添加`lib`库文件  

   库文件是交叉编译器的库文件。
   
   - 将`/usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/arm-linux-
     gnueabihf/libc/lib`，`/usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/arm-linux-gnueabihf/lib`下的内容拷贝到`rootfs`下的`lib`文件夹内。  
   
   - 重新拷贝`ld-linux-armhf.so.3`文件到`rootfs`下的`lib`中，而不是上面拷贝的软链接文件。
   
     ```c
     cp ld-linux-armhf.so.3 /home/zuozhongkai/linux/nfs/rootfs/lib/`
     ```
   
   - 拷贝`/usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/arm-linux-gnueabihf/libc/usr/lib`下的文件到`rootfs/usr/lib`下。  
   
   - 最后在根文件系统中创建其他文件夹,如 `dev`、`proc`、`mnt`、`sys`、`tmp` 和 `root` 等。

## 3.根文件系统初步测试

​	为了方便开发测试，使用`NFS`挂载测试。Linux内核文件中有说明命令行参数如何设置。要求：

1. linux的内核网络驱动要工作正常。
2. 设置uboot的`bootargs`，也就是linux内核的命令行参数。其中`ip`的参数为`本机地址:服务器地址:网关:子网掩码:[主机名]:网卡:自动配置`

```
setenv bootargs 'console=ttymxc0,115200 root=/dev/nfs nfsroot=192.168.1.102:/home/rabbit/linux/nfs/rootfs,proto=tcp rw ip=192.168.1.170:192.168.1.102:192.168.1.1:255.255.255.0::eth0:off'
```

==如果挂载失败，显示如下信息==  

```
VFS: Unable to mount root fs via NFS, trying floppy.
VFS: Cannot open root device "nfs" or unknown-block(2,0): error -6
```

修改`bootargs` 环境变量为以下的值。

```
setenv bootargs 'console=ttymxc0,115200 root=/dev/nfs nfsroot=192.168.1.102:/home/rabbit/linux/nfs/rootfs,nfsvers=3,proto=tcp rw ip=192.168.1.170:192.168.1.102:192.168.1.1:255.255.255.0::eth0:off'
```

## 4. 完善根文件系统   

由于出现以下错误，因此需要完善根文件系统。

```
can't run '/etc/init.d/rcS': No such file or directory
```

1. 创建`/etc/init.d/rcS`  写入以下内容

```sh
#!/bin/sh

PATH=/sbin:/bin:/usr/sbin:/usr/bin:${PATH}
LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:/lib:/usr/lib
export PATH LD_LIBRARY_PATH

mount -a
mkdir /dev/pts
mount -t devpts devpts /dev/pts

echo /sbin/mdev > /proc/sys/kernel/hotplug
mdev -s
```

2. 创建`/etc/fstab`文件，写入以下内容  

```
#<file system> 	<mount point>	<type>  <options>       <dump>  <pass>
proc            /proc           proc    defaults        0       0
tmpfs           /tmp            tmpfs   defaults        0       0
sysfs           /sys            sysfs   defaults        0       0
tmpfs           /dev            tmpfs   defaults        0       0
```

3. 创建`/etc/inittab` 文件，写入如下内容  

```
#/etc/initable
::sysinit:/etc/init.d/rcS
console::askfirst:-/bin/sh
::restart:/sbin/init
::ctrlaltdel:/sbin/reboot
::shutdown:/bin/umount -a -r
::shutdowm:/sbin/swapoff -a
```

## 5. 测试  

1. 写个小程序测试下  

`hello.c`

```c
#include <stdio.h>

int main(void)
{
	while(1)
	{
		printf("hello-world\r\n");
		sleep(2);
	}
	return 0;
}
```

因为是运行在`ARM`上，所以需要使用交叉编译器编译这个`.c`文件，可以使用`file`命令来查看信息。

```
arm-linux-gnueabihf-gcc hello.c -o hello
```

2. 开机自启动

在`/etc/init.d/rcS`添加开机自启动程序。

```sh
#开机自启动
/root/hello &
```

3. 设置域名解析服务器地址（DNS）

   新建`/etc/resolv.conf`，写入以下内容  

```
nameserver 114.114.114.114
nameserver 192.168.1.1
```

此时就可以`ping www.baidu.com`了。