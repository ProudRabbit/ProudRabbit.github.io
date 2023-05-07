---
title: IMX6ULL嵌入式Linux驱动学习笔记（二）
date: 2020-09-19 21:40:24
tags: [嵌入式, IMX6ULL, 学习笔记]
keywords:
description: 正点原子alpha开发板IMX6ULL嵌入式Linux开发学习笔记。
---

**IMX6ULL嵌入式Linux驱动开发学习**

以下内容是我在学习正点原子`IMX6ULL`开发板`alpha`中记录的笔记，部分摘录自正点原子`IMX6ULL开发手册`。

## 一、字符设备驱动

字符设备驱动的编写主要就是驱动对应的`open`、`close`、`read`、`write`函数。其实就是file_operations结构体的成员变量的实现。

## 二、驱动模块的加载与卸载

Linux驱动程序可以编译到kernel里面，也就是zImage，也可以编译为模块(.ko)。测试的时候只需要加载.ko模块就行。

- `module_init(xxx_init);`   //注册模块加载函数
- `module_exit(xxx_exit)`   //注册模块卸载函数  

<!--more-->

编写驱动的时候注意事项！

1. 编译驱动的时候需要用到`linux`内核源码！因此需要解压缩`linux`源码，编译`linux`内核源码。得到`zImage`和`dtb`。需要使用编译后得到的`zImage`和`dtb`启动系统。

   `vscode`中设置`linux`源码所在路径，`.vscode/c_cpp_properties.json`：

   ```json
   {
       "configurations": [
           {
               "name": "Linux",
               "includePath": [
                   "${workspaceFolder}/**",
                   "/home/rabbit/linux/IMX6UL/linux_image/linux-imx-alientek/include", 
                   "/home/rabbit/linux/IMX6UL/linux_image/linux-imx-alientek/arch/arm/include", 
                   "/home/rabbit/linux/IMX6UL/linux_image/linux-imx-alientek/arch/arm/include/generated/"
               ],
               "defines": [],
               "compilerPath": "/usr/bin/gcc",
               "cStandard": "c11",
               "cppStandard": "c++17",
               "intelliSenseMode": "clang-x64"
           }
       ],
       "version": 4
   }
   ```

   `makefile`内容

   ```makefile
   # 内核路径
   KERNELDIR := /home/rabbit/linux/IMX6UL/linux_image/linux-imx-alientek
   
   # 当前路径
   CURRENT_PATH := $(shell pwd)
   
   # 目标文件
   obj-m := chrdevbase.o
   
   # 规则
   build : kernel_modules
   
   kernel_modules:
   	$(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) modules
   
   clean:
   	$(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) clean
   ```

2. 将编译出来的`.ko`文件放到根文件系统中。加载驱动会用到加载命令：`insmod`，`modprobe`。移除驱动使用命令：`rmmod`，查看加载的驱动模块命令：`lsmod`。

   - `insmod`：不会解决模块的依赖关系。
   - `modprobe`：可以处理模块的依赖关系。**推荐使用**，`modprobe`会到`/lib/modules/内核版本`下查找相应的驱动模块。  

   ==对于一个新的模块使用`modprode`加载的时候需要先调用一下`depmod`命令来分析可载入模块的相依性。==

## 三、字符设备的注册与注销  

1. 我们需要向系统注册一个字符设备，使用函数（即将弃用）：

   ```c
   static inline init register_chrdev(unsigned int major, const char *name, const struct file_operations *fops)
   ```

2. 卸载驱动的时候需要注销掉前面注册的字符设备，使用函数（即将弃用）：

   ```c
   static inline void unregister_chrdev(unsigned int major, const char *name)
   ```

   - `major`：主设备号，linux下每个设备都有一个设备号，设备号分为主设备号和次设备号两个部分。传入`0`自动分配。
   - `name`：设备名字，指向一串字符串。
   - `fops`：结构体`file_operations`类型指针，指向设备的操作函数集合变量。

## 四、设备号  

1. linux内核使用`dev_t`

   ```c
   typedef __kernel_dev_t		dev_t;
   typedef __u32 __kernel_dev_t;
   typedef unsigned int __u32;
   ```

   其中 `dev_t` 是一个无符号32位整型数据，其中高12位为主设备号(0～4096，表示同一类设备，比如IIC设备)，低20位为次设备号。

2. 设备号的操作函数或宏

   从`dev_t`获取主设备号和次设备号，`MAJOR(dev_t)`，`MINOR(dev_t)`，也可以使用主设备号和次设备号构成`dev_t`，通过`MKDE(major, minor)`即可。

## 五、file_operations的具体实现   

```c
struct file_operations {
	struct module *owner;
	loff_t (*llseek) (struct file *, loff_t, int);
	ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
	ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
	ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
	ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);
	int (*iterate) (struct file *, struct dir_context *);
	unsigned int (*poll) (struct file *, struct poll_table_struct *);
	long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
	long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
	int (*mmap) (struct file *, struct vm_area_struct *);
	int (*mremap)(struct file *, struct vm_area_struct *);
	int (*open) (struct inode *, struct file *);
	int (*flush) (struct file *, fl_owner_t id);
	int (*release) (struct inode *, struct file *);
	int (*fsync) (struct file *, loff_t, loff_t, int datasync);
	int (*aio_fsync) (struct kiocb *, int datasync);
	int (*fasync) (int, struct file *, int);
	int (*lock) (struct file *, int, struct file_lock *);
	ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int);
	unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);
	int (*check_flags)(int);
	int (*flock) (struct file *, int, struct file_lock *);
	ssize_t (*splice_write)(struct pipe_inode_info *, struct file *, loff_t *, size_t, unsigned int);
	ssize_t (*splice_read)(struct file *, loff_t *, struct pipe_inode_info *, size_t, unsigned int);
	int (*setlease)(struct file *, long, struct file_lock **, void **);
	long (*fallocate)(struct file *file, int mode, loff_t offset,
			  loff_t len);
	void (*show_fdinfo)(struct seq_file *m, struct file *f);
#ifndef CONFIG_MMU
	unsigned (*mmap_capabilities)(struct file *);
#endif
};
```

## 六、字符设备驱动框架  

多借鉴别人的驱动程序。

```c
#include <linux/types.h>
#include <linux/kernel.h>
#include <linux/delay.h>
#include <linux/ide.h>
#include <linux/init.h>
#include <linux/module.h>
#include <linux/gpio.h>
#include <linux/cdev.h>
#include <linux/of.h>
#include <linux/of_address.h>
#include <linux/of_irq.h>

#define CHRDEVBASE_NAME		"chrdevbase"	// 名字
#define CHRDEVBASE_CNT		1

/* 设备结构体 */
struct chrdev 
{
	dev_t devid;			/* 设备号 */
	int major;				/* 主设备号 */
    int minor;
	struct cdev cdev;		/* 字符设备 */
	struct class *class;	/* 类 */
	struct device *device;	/* 设备节点 */
	struct device_node *nd;	/* 设备树节点 */
};

struct chrdev chrdevbase;

static int chrdevbase_open(struct inode *inode, struct file *filp)
{
    filp->private_data = &chrdevbase;
	printk("chrdevbase_open!");
	return 0;
}

static int chrdevbase_close(struct inode *inode, struct file *filp)
{
    struct chrdev *dev = filp->private_data;
	printk("chrdevbase_close!");
	return 0;
}

static ssize_t chrdevbase_read(struct file *filp, char __user *buf, size_t count, loff_t *ppos)
{
    struct chrdev *dev = filp->private_data;
	printk("chrdevbase_read!");
	return 0;
}

static ssize_t chrdevbase_write(struct file *filp, const char __user *buf, size_t count, loff_t *ppos)
{
    struct chrdev *dev = filp->private_data;
	printk("chrdevbase_write!");
	return 0;
}

static struct file_operations chrdevbase_fops = {
	.owner = THIS_MODULE,
	.open = chrdevbase_open,
	.release = chrdevbase_close,
	.read = chrdevbase_read,
	.write = chrdevbase_write,
};

static int __init chrdevbase_init(void)
{
	int ret = 0;

	/* 1.注册字符设备 */
	/* 1.1 申请设备号 */
	chrdevbase.major = 0;	/* 设备号由内核分配 */
	if (chrdevbase.major)
	{
		/* 定义了设备号 */
		chrdevbase.devid = MKDEV(chrdevbase.major,0);
		ret = register_chrdev_region(chrdevbase.devid, CHRDEVBASE_CNT, CHRDEVBASE_NAME);
	}
	else
	{
		/* 没有给定设备号,向内核申请*/
		ret = alloc_chrdev_region(&chrdevbase.devid, 0, CHRDEVBASE_CNT, CHRDEVBASE_NAME);
	}
	
	if (ret < 0)
	{
		goto fail_devid;
	}

	/* 1.1 添加字符设备 */
	chrdevbase.cdev.owner = THIS_MODULE;
	cdev_init(&chrdevbase.cdev, &chrdevbase_fops);
	ret = cdev_add(&chrdevbase.cdev, chrdevbase.devid, CHRDEVBASE_CNT);
	if (ret < 0)
	{
		goto fail_cdev;
	}
	
	/* 3.自动创建设备节点 */
	/* 3.1 创建类 */
	chrdevbase.class = class_create(THIS_MODULE, CHRDEVBASE_NAME);
	if (IS_ERR(chrdevbase.class))
	{
		ret = PTR_ERR(chrdevbase.class);
		goto fail_class;
	}

	/* 3.2创建设备节点 */
	chrdevbase.device = device_create(chrdevbase.class, NULL, chrdevbase.devid, NULL, CHRDEVBASE_NAME);
	if (IS_ERR(chrdevbase.device))
	{
		ret = PTR_ERR(chrdevbase.device);
		goto fail_device;
	}

	return 0;

fail_findnd:
	device_destroy(chrdevbase.class, chrdevbase.devid);
fail_device:
	class_destroy(chrdevbase.class);
fail_class:
	cdev_del(&chrdevbase.cdev);
fail_cdev:
	unregister_chrdev_region(chrdevbase.devid, CHRDEVBASE_CNT);
fail_devid:
	return ret;
}

static void __exit chrdevbase_exit(void)
{
	/* 摧毁设备节点 */
	device_destroy(chrdevbase.class, chrdevbase.devid);
	/* 摧毁类 */
	class_destroy(chrdevbase.class);
	/* 删除字符设备 */
	cdev_del(&chrdevbase.cdev);
	/* 释放设备号 */
	unregister_chrdev_region(chrdevbase.devid, CHRDEVBASE_CNT);
}

/**
 * 模块入口与出口
 * */
module_init(chrdevbase_init);	// 入口函数
module_exit(chrdevbase_exit);	// 出口函数
MODULE_LICENSE("GPL");
MODULE_AUTHOR("XXXX");
```

## 七、编写应用程序

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>

/**
 * ./chrdevbaseAPP <filename>
 * @param argc 应用程序参数个数
 * @param argv 保存的参数，字符串形式。
 * */
int main(int argc, char *argv[])
{
	int ret = 0;
	int fd = 0;
	char *filename;
	char readbuf[100];
	char writebuf[100];

	filename = argv[1];

	fd = open(filename, O_RDWR);
	if(fd < 0) {
		printf("can't open file %s\r\n",filename);
		return -1;
	}

	/* 读 */
	ret = read(fd, readbuf, 10);
	if (ret < 0)
	{
		printf("read file %s failed\r\n", filename);
	}
	else
	{
		/* code */
	}
	/* 写 */
	ret = write(fd, writebuf, 50);
	if (ret < 0)
	{
		printf("write file %s failed\r\n", filename);
	}

	/* 关闭 */
	ret = close(fd);
	if (ret < 0)
	{
		printf("close file %s failed\r\n", filename);
	}
	return 0;
}
```

## 八、测试  

1. 加载驱动

   ```
   modprobe chrdevbase.ko
   ```

2. 进入`/dev`查看设备文件，`chrdevbase`。但是由于没有创建设备节点`/dev/chrdevbase`并不会存在。这里使用`mknod /dev/chardevbase c 100 0`手动创建设备节点。

3. 测试

   `./chrdevbaseAPP /dev/chrdevbase`

## 九、完善chrdevbase虚拟字符设备驱动程序  

- 驱动给应用传递数据的时候需要用到`copy_to_user(to, from, n)`函数；
- 应用给驱动传递数据的时候需要用到`copy_from_user(to, from, n)`函数；

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/fs.h>
#include <linux/uaccess.h>

#define CHRDEVBASE_MAJOR	100				// 主设备号，0自动分配
#define CHRDEVBASE_NAME		"chrdevbase"	// 名字

static char readbuf[100];
static char writebuf[100];
static char kerneldata[] = {"kernel data!"};

static int chrdevbase_open(struct inode *inode, struct file *filp)
{
	printk("chrdevbase_open!\r\n");
	return 0;
}

static int chrdevbase_close(struct inode *inode, struct file *filp)
{
	printk("chrdevbase_close!\r\n");
	return 0;
}

static ssize_t chrdevbase_read(struct file *filp, char __user *buf, size_t count, loff_t *ppos)
{
	int ret = 0;
	/* printk("chrdevbase_read!\r\n"); */
	ret = copy_to_user(buf, kerneldata, count);
	if (ret == 0)
	{
		/* code */
	}
	else
	{
		/* code */
	}

	return 0;
}

static ssize_t chrdevbase_write(struct file *filp, const char __user *buf, size_t count, loff_t *ppos)
{
	int ret = 0;
	/* printk("chrdevbase_write!\r\n"); */
	ret = copy_from_user(writebuf, buf, count);
	if (ret == 0)
	{
		printk("kernel recevdata:%s\r\n",writebuf);
	}
	else
	{
		/* code */
	}
	
	return 0;
}

static struct file_operations chrdevbase_fops = {
	.owner = THIS_MODULE,
	.open = chrdevbase_open,
	.release = chrdevbase_close,
	.read = chrdevbase_read,
	.write = chrdevbase_write,
};

static int __init chrdevbase_init(void)
{
	int ret = 0;
	printk("chrdevbase_init\r\n");
	/* 注册字符设备 */
	ret = register_chrdev(CHRDEVBASE_MAJOR, CHRDEVBASE_NAME, &chrdevbase_fops);
	if (ret < 0)
	{
		printk("chrdevbase_init failed\r\n");
	}
	return 0;
}

static void __exit chrdevbase_exit(void)
{
	printk("chrdevbase_exit\r\n");
	unregister_chrdev(CHRDEVBASE_MAJOR, CHRDEVBASE_NAME);
	/* 注销字符设备 */
}

/**
 * 模块入口与出口
 * */
module_init(chrdevbase_init);	// 入口函数
module_exit(chrdevbase_exit);	// 出口函数

MODULE_LICENSE("GPL");
```

