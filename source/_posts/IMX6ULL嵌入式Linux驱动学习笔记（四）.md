---
title: IMX6ULL嵌入式Linux驱动学习笔记（四）
date: 2020-09-19 21:48:02
tags: [嵌入式, IMX6ULL, 学习笔记]
keywords:
description: 正点原子alpha开发板IMX6ULL嵌入式Linux开发学习笔记。
---

**IMX6ULL嵌入式Linux驱动开发学习**

以下内容是我在学习正点原子`IMX6ULL`开发板`alpha`中记录的笔记，部分摘录自正点原子`IMX6ULL开发手册`。

## 一、新字符设备驱动原理（相比于上一篇笔记）

1. 以前的缺点：

   使用 `register_chrdev` 函数注册字符设备，会浪费很多次设备号，而且需要手动指定。

2. 新的方法：

   使用 `alloc_chrdev_region` 函数申请设备号。原型如下：

   <!--more-->

```c
int alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count, const char *name)
```

   卸载驱动的时候，使用 `unregister_chrdev_region` 函数释放前面申请的设备号，原型如下：

```c
void unregister_chrdev_region(dev_t from, unsigned count)
```

3. 指定设备号

   如果指定主设备号，使用 `register_chrdev_region` 函数来注册，原型如下：

```c
int register_chrdev_region(dev_t from, unsigned count, const char *name)
```

   	一般是给定主设备号，然后使用 `MADEV` 构建完整的 `dev_t` ，一般次设备号选择`0`。

4. 实际的驱动编写

   需要考虑实际情况，因为在实际开发中会有两种情况：给定设备号和没有给定设备号。

5. 字符设备注册

   `cdev` 结构体表示字符设备，然后使用 `cdev_init` 函数来初始化 `cdev` ，原型

```c
cdev_init(struct cdev *cdev, const struct file_operations *fops)
```

​		`cdev_init`  初始化完成 `cdev` 后，使用 `cdev_add` 添加到Linux内核，删除字符设备使用 `cdev_del` 。



## 二、自动创建设备节点  

1. linux内核在2.6版本中引入了 `udev` 机制，替换`devfs` 。`udev` 机制提供热插拔管理，可以在加载驱动的时候自动创建 `/dev/xxx` 设备文件。
2. 在使用busybox构建根文件系统的时候，busybox会创建一个 `udev` 的简化版本 `mdev` ，所以在嵌入式linux中使用 `mdev` 来实现设备节点文件的自动创建和删除。linux系统中的热插拔事件也由 `mdev` 管理，在 `/etc/init.d/rsS` 文件中如下语句。

```sh
11 echo /sbin/mdev > /proc/sys/kernel/hotplug
12 mdev -s
```

3. 创建设备节点
   3.1. 创建设备类

   创建设备节点需要先使用`class_create()` 函数创建类。
   
   ```c
   /* 创建类 */
   newChrLed.class = class_create(THIS_MODULE, LED_NAME);
   if (IS_ERR(newChrLed.class))
   {
       return PTR_ERR(newChrLed.class);
   }
   ```
   
   3.2. 创建设备节点
   
   使用 `device_create()` 函数来创建设备节点。
   
   ```c
   /* 创建设备节点 */
   newChrLed.device = device_create(newChrLed.class, NULL, newChrLed.devid, NULL, LED_NAME);
   if (IS_ERR(newChrLed.device))
   {
       return PTR_ERR(newChrLed.device);
   }
   ```

4. 销毁设备节点和类

   在驱动卸载的时候，需要对设备节点进行销毁。在驱动出口函数中使用如下代码进行销毁：
   
   ```c
   /* 摧毁设备节点 */
   device_destroy(newChrLed.class, newChrLed.devid);
   
   /* 摧毁类 */
   class_destroy(newChrLed.class);
   ```

​		**因为创建设备节点是根据类来创建的，因此在销毁时，需要先销毁设备节点再销毁类。**

## 三、文件私有数据  

1. 在 `open` 函数里设置 `filp->private_data` 为设备变量。

```c
/* 设置私有数据 */
filp->private_data= &newChrLed;
```

2. 在其他的函数里，要访问设备的时候，直接读取私有数据

```c
struct newChrLed_dev *dev = filp->private_data;
```

## 四、错误处理

​	在 `xxx_init` 加载驱动出现错误的时候，可以使用 `goto` 语句，对错误进行处理。比如：

```c
newChrLed.cdev.owner = THIS_MODULE;
ret = cdev_init(&newChrLed.cdev, &newChrLed_fops);
if (ret < 0)
{
    goto fail_cdev;
}

ret = cdev_add(&newChrLed.cdev, newChrLed.devid, 1);
if (ret < 0)
{
    goto fail_cdev;
}

fail_cdev:
	/* 删除字符设备 */
	cdev_del(&newChrLed.cdev);
	/* 因为cdev初始化失败，所以需要注销设备号 */
	unregister_chrdev_region(newChrLed.devid, 1);
	return -1;
```

## 五、整体程序（开发中不使用这种方式）

```c
#include <linux/types.h>
#include <linux/kernel.h>
#include <linux/delay.h>
#include <linux/ide.h>
#include <linux/init.h>
#include <linux/module.h>
#include <linux/errno.h>
#include <linux/gpio.h>
#include <asm/mach/map.h>
#include <asm/uaccess.h>
#include <asm/io.h>
#include <linux/cdev.h>

#define LED_NAME	"LED"

/* 寄存器物理地址 */
#define CCM_CCGR1_BASE			(0X020C406C)
#define SW_MUX_GPIO1_IO03_BASE	(0X020E0068)
#define SW_PAD_GPIO1_IO03_BASE	(0X020E02F4)
#define GPIO1_GDIR_BASE			(0X0209C004)
#define GPIO1_DR_BASE			(0X0209C000)

/* 虚拟地址的指针 */
static void __iomem *CCM_CCGR1;
static void __iomem *SW_MUX_GPIO1_IO03;
static void __iomem *SW_PAD_GPIO1_IO03;
static void __iomem *GPIO1_GDIR;
static void __iomem *GPIO1_DR;

#define LEDOFF	0
#define LEDON 	1

/* LED设备结构体 */
struct newChrLed_dev
{
	struct cdev cdev;		// cdev
	dev_t devid;			// 设备号
	struct class *class;	// 类
	struct device *device;	// 设备
	int major;				// 主设备号
	int minor;				// 次设备号
};

static struct newChrLed_dev newChrLed;	// LED设备

void led_toggle(u8 state)
{
	u32 val = 0;

	if(state == LEDON)
	{
		val = readl(GPIO1_DR);
		val &= ~(1 << 3);
		writel(val, GPIO1_DR);	// 打开LED
	}
	else
	{
		val = readl(GPIO1_DR);
		val |= (1 << 3);
		writel(val, GPIO1_DR);	// 关闭LED
	}
	
}

static int newchrled_open(struct inode *inode, struct file *filp)
{
	/* 设置私有数据 */
	filp->private_data= &newChrLed;
	return 0;
}

static int newchrled_release(struct inode *inode, struct file *filp)
{
	filp->private_data = NULL;
	return 0;
}

static ssize_t newchrled_write(struct file *filp, const char __user *buf, size_t len, loff_t * off)
{
	int retvalue;
	unsigned char databuff[1];
	// struct newChrLed_dev *dev = filp->private_data;
	retvalue = copy_from_user(databuff, buf, len);
	if (retvalue < 0)
	{
		printk("kernel write failed!\r\n");
		return -EFAULT;
	}

	if (databuff[0] == LEDON)
	{
		led_toggle(LEDON);
	}
	else
	{
		led_toggle(LEDOFF);
	}
	
	return 0;
}

struct file_operations newChrLed_fops = {
	.owner = THIS_MODULE,
	.write = newchrled_write,
	.open = newchrled_open,
	.release = newchrled_release,
};

/**
 * 入口
 * */
static int __init newchrled_init(void)
{
	int ret = 0;
	/* 1.初始化LED */
	unsigned int val = 0;
	/* 初始化LED灯，地址映射 */
	CCM_CCGR1 = ioremap(CCM_CCGR1_BASE, 4);
	SW_MUX_GPIO1_IO03 = ioremap(SW_MUX_GPIO1_IO03_BASE, 4);
	SW_PAD_GPIO1_IO03 = ioremap(SW_PAD_GPIO1_IO03_BASE, 4);
	GPIO1_GDIR = ioremap(GPIO1_GDIR_BASE, 4);
	GPIO1_DR = ioremap(GPIO1_DR_BASE, 4);

	/* 初始化 */
	val = readl(CCM_CCGR1);
	val &= ~(3 << 26);
	val |= (3 << 26);
	writel(val, CCM_CCGR1);	// 使能时钟

	writel(0x5, SW_MUX_GPIO1_IO03);		// 设置复用
	writel(0x10b0, SW_PAD_GPIO1_IO03);	// 设置电气属性
	
	val = readl(GPIO1_GDIR);
	val |= (1 << 3);
	writel(val, GPIO1_GDIR);	// 设置输出

	val = readl(GPIO1_DR);
	val &= ~(1 << 3);
	writel(val, GPIO1_DR);	// 默认打开LED

	newChrLed.major = 0;	// 设置为0，表示由系统自动分配设备号
	/* 2.注册字符设备 */
	if (newChrLed.major)
	{
		// 给定主设备号
		newChrLed.devid = MKDEV(newChrLed.major, 0);
		ret = register_chrdev_region(newChrLed.devid, 1, LED_NAME);
	}
	else
	{
		// 没有给定主设备号
		ret = alloc_chrdev_region(&newChrLed.devid, 0, 1, LED_NAME);
		newChrLed.major = MAJOR(newChrLed.devid);
		newChrLed.minor = MINOR(newChrLed.devid);
	}

	if (ret < 0)
	{
		printk("newchrled chrdev_region error!\r\n");
		goto fail_devid;
	}

	printk("newchrled major = %d, minor = %de\r\n", newChrLed.major, newChrLed.minor);
	
	/* 3.字符设备注册 */
	newChrLed.cdev.owner = THIS_MODULE;
	cdev_init(&newChrLed.cdev, &newChrLed_fops);
	ret = cdev_add(&newChrLed.cdev, newChrLed.devid, 1);
	if (ret < 0)
	{
		goto fail_cdev;
	}
	
	/* 4.自动创建设备节点 */
	/* 创建类 */
	newChrLed.class = class_create(THIS_MODULE, LED_NAME);
	if (IS_ERR(newChrLed.class))
	{
		ret = PTR_ERR(newChrLed.class);
		goto fail_class;
	}
	/* 创建设备节点 */
	newChrLed.device = device_create(newChrLed.class, NULL, newChrLed.devid, NULL, LED_NAME);
	if (IS_ERR(newChrLed.device))
	{
		ret = PTR_ERR(newChrLed.device);
		goto fail_device;
	}

	return 0;

fail_device:
	class_destroy(newChrLed.class);

fail_class:
	/* 删除字符设备 */
	cdev_del(&newChrLed.cdev);

fail_cdev:
	/* 因为cdev初始化失败，所以需要注销设备号 */
	unregister_chrdev_region(newChrLed.devid, 1);

fail_devid:
	return ret;
}

/**
 * 出口
 * */
static void __exit newchrled_exit(void)
{
    /* 取消地址映射 */
	iounmap(IMX6U_CCM_CCGR1);
	iounmap(SW_MUX_GPIO1_IO03);
	iounmap(SW_PAD_GPIO1_IO03);
	iounmap(GPIO1_DR);
	iounmap(GPIO1_GDIR);
    
	/* 删除字符设备 */
	cdev_del(&newChrLed.cdev);

	/* 注销设备号 */
	unregister_chrdev_region(newChrLed.devid, 1);

	/* 摧毁设备 */
	device_destroy(newChrLed.class, newChrLed.devid);

	/* 摧毁类 */
	class_destroy(newChrLed.class);
}

/**
 * 注册入口和出口
 * */
module_init(newchrled_init);
module_exit(newchrled_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("fengyuhang");
```

## 六、测试应用程序  

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

/**
 * ./ledAPP <filename> <0:1> 1 表示开灯 0 表示关灯
 * @param argc 应用程序参数个数
 * @param argv 保存的参数，字符串形式。
 * */
int main(int argc, char *argv[])
{
	int ret = 0;
	int fd = 0;
	char *filename;
	unsigned char databuff[1];

	if ( argc !=3)
	{
		printf("输入错误\r\n");
	}
	
	filename = argv[1];

	fd = open(filename, O_RDWR);
	if(fd < 0) {
		printf("Error: %s\r\n",filename);
		return -1;
	}

	/* 读 */
/* 	if (atoi(argv[2]) == 1)		// 传递过来的是字符串，需要转换成数字
	{
		ret = read(fd, readbuf, 10);
		if (ret < 0)
		{
			printf("read file %s failed\r\n", filename);
		}
		else
		{
			
		}
	} */
	
	databuff[0] = atoi(argv[2]);

	/* 写 */
	ret = write(fd, databuff, sizeof(databuff));
	if (ret < 0)
	{
		printf("LED control failed!\r\n");
		close(fd);
		return -1;
	}

	/* 关闭 */
	ret = close(fd);
	
	return 0;
}
```

