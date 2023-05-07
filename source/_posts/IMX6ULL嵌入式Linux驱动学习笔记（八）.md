---
title: IMX6ULL嵌入式Linux驱动学习笔记（八）
date: 2020-11-17 22:51:31
tags: [嵌入式, IMX6ULL, 学习笔记]
keywords:
description: 正点原子alpha开发板IMX6ULL嵌入式Linux开发学习笔记。
---

**IMX6ULL嵌入式Linux驱动开发学习**

以下内容是我在学习正点原子`IMX6ULL`开发板`alpha`中记录的笔记，部分摘录自正点原子`IMX6ULL开发手册`。

# Linux内核定时器和ioctl函数

1. 内核时间管理

   1.1  对于`Cortex-M`内核来说，一般使用`systick`硬件定时器作为系统定时器，使用过`FreeRTOS`等操作系统的也知道，`FreeRTOS`一般就是使用`systick`来提供系统时钟，同样`Linux`也需要系统时钟。

   1.2  `Linux`内核频率可以配置，在图形化界面可以配置。在`include/aasm-generic/param.h`文件中可以看到内核配置的节拍率`HZ`（系统频率）。

   1.3 `HZ`表示每秒的节拍数。

   <!--more-->

2. 节拍率高低的缺陷

   系统使用硬件定时中断来计时，中断周期性产生的频率就是系统频率，也叫作节拍率`tick rate`。

   节拍率越高，时间精度越高，同样也会加剧系统的负担。

3. `jiffies`

   `Linux`内核使用全局变量`jiffies`来记录系统从启动以来的系统节拍数，系统启动的时候会将其初始化为0，`jiffies`定义在文件`include/linux/jiffies.h`中。`jiffies`/`HZ`就是系统运行时间，单位为秒。

```c
/************** 判断是否超时的api函数  ***************/ 

/* unkown 通常为 jiffies， known 通常是需要对比的值。 */
time_after(unkown, known)
time_before(unkown, known) 	
time_after_eq(unkown, known)
time_before_eq(unkown, known)


/************** jiffies 和 ms、us、ns的转化函数  ***************/ 

/* 将 jiffies 类型的参数 j 分别转换为对应的毫秒、微秒、纳秒。 */
int jiffies_to_msecs(const unsigned long j)
int jiffies_to_usecs(const unsigned long j) 
u64 jiffies_to_nsecs(const unsigned long j)

/* 将毫秒、微秒、纳秒转换为 jiffies 类型。 */
long msecs_to_jiffies(const unsigned int m)
long usecs_to_jiffies(const unsigned int u) 
unsigned long nsecs_to_jiffies(u64 n)

/***************** 使用 jiffies 判断超时 ****************/ 
    
unsigned long timeout;
timeout = jiffies + (2 * HZ); /* 超时的时间点 */

/* 判断有没有超时 */
if(time_before(jiffies, timeout)) 
{
	/* 超时未发生 */
} 
else
{
	/* 超时发生 */
}
```

4. 内核定时器

   4.1 软件定时器不像硬件定时器一样是直接设置周期值的。软件定时器是设置期限满以后的时间点。

   4.2 需要编写定时器处理函数。

   4.3 内核定时器不是周期性的，一次定时时间到了以后就会关闭，除非重新打开。

5. 定时器`API`函数

```c
/******** timer_list 结构体 **********/
struct timer_list {
    struct list_head entry;
    unsigned long expires; 				/* 定时器超时时间，单位是节拍数 */
    struct tvec_base* base;
    void (*function)(unsigned long);	/* 定时处理函数 */
    unsigned long data; 				/* 要传递给 function 函数的参数 */
    int slack;
};

/************ api函数 ************/
void init_timer(struct timer_list* timer);		// 初始化定时器
void add_timer(struct timer_list* timer);		// 向Linux内核注册定时器，同时启动定时器
int del_timer(struct timer_list* timer);		// 删除一个定时器
int del_timer_sync(struct timer_list* timer);	// del_timer 函数的同步版，会等待其他处理器使用完定时器再删除，del_timer_sync 不能使用在中断上下文中
int mod_timer(struct timer_list* timer, unsigned long expires);	// 修改定时值，同时启动定时器
```

6. 内核短延时函数

```c
void ndelay(unsigned long nsecs);	// 纳秒延时
void udelay(unsigned long usecs);	// 微秒延时
void mdelay(unsigned long mseces);	// 毫秒延时
```

# 定时器使用练手

```c
#include <linux/types.h>
#include <linux/kernel.h>
#include <linux/delay.h>
#include <linux/ide.h>
#include <linux/init.h>
#include <linux/module.h>
#include <linux/errno.h>
#include <linux/gpio.h>
#include <linux/cdev.h>
#include <linux/of.h>
#include <linux/of_address.h>
#include <linux/of_irq.h>
#include <linux/of_gpio.h>
#include <linux/timer.h>
#include <linux/jiffies.h>
#include <asm/mach/map.h>
#include <asm/uaccess.h>
#include <asm/io.h>


#define TIMER_CNT	1
#define TIMER_NAME	"timer"

#define LEDOFF	0
#define LEDON 	1

/* 定义设备驱动结构体 */
struct timer_dev
{
	dev_t devid;
	int major;
	int minor;
	struct cdev cdev;
	struct class *class;
	struct device *device;
	struct device_node *nd;
	int led_gpio;				/* led 所使用的 GPIO 编号 */
	struct timer_list timer;	/* 定义一个定时器 */
	
};

struct timer_dev timerdev;		/* 定义一个设备 */

static int led_open(struct inode *inode, struct file *filp)
{
	filp->private_data = &timerdev;
	
	return 0;
}	

static int led_release(struct inode *inode, struct file *filp)
{
	struct timer_dev *dev = filp->private_data;
	return 0;
}

ssize_t led_write(struct file *filp, const char __user *buf, size_t count, loff_t *ppos)
{
	struct timer_dev *dev = filp->private_data;
	unsigned char databuff[1];
	int ret;

	ret = copy_from_user(databuff, buf, count);
	if (ret < 0)
	{
		printk("kernel write failed!\r\n");
		return -EFAULT;
	}
	if(databuff[0] == LEDON)
	{
		gpio_set_value(dev->led_gpio, 0);
	}
	else
	{
		gpio_set_value(dev->led_gpio, 1);
	}
	
	return 0;
}

struct file_operations led_fops = {
	.owner = THIS_MODULE,
	.open = led_open,
	.release = led_release,
	.write = led_write,
};

static void timer_function(unsigned long arg)
{
	static int i = 0;
	mod_timer(&timerdev.timer, jiffies + msecs_to_jiffies(500));	// 修改时间点，并重新开启定时器。
	gpio_set_value(timerdev.led_gpio, i);
	i = !i;
}

/* 入口和出口 */
static int __init timer_init(void)
{
	int ret = 0;
	
	/* 注册字符设备ID */
	timerdev.major = 0;
	if(timerdev.major)
	{
		/* 给定设备号 */
		timerdev.devid = MKDEV(timerdev.major, 0);
		ret = register_chrdev_region(timerdev.devid, TIMER_CNT, TIMER_NAME);
	}
	else
	{
		alloc_chrdev_region(&timerdev.devid, 0,TIMER_CNT,TIMER_NAME);
	}

	if (ret < 0)
	{
		goto fail_devid;
	}

	/* 初始化字符设备 */
	timerdev.cdev.owner = THIS_MODULE;
	cdev_init(&timerdev.cdev, &led_fops);
	ret = cdev_add(&timerdev.cdev, timerdev.devid, TIMER_CNT);
	if (ret < 0)
	{
		goto fail_cdev;
	}

	/* 创建类 */
	timerdev.class = class_create(THIS_MODULE, TIMER_NAME);
	if (IS_ERR(timerdev.class))
	{
		ret = PTR_ERR(timerdev.class);
		goto fail_class;
	}

	/* 创建设备节点 */
	timerdev.device = device_create(timerdev.class, NULL, timerdev.devid, NULL, TIMER_NAME);
	if (IS_ERR(timerdev.device))
	{
		ret = PTR_ERR(timerdev.device);
		goto fail_device;
	}

	/* 获取设备节点 */
	timerdev.nd = of_find_node_by_path("/gpioled");
	if (timerdev.nd == NULL)
	{
		ret = -EINVAL;
		goto fail_findnd;
	}

	/* 获取LED对应的GPIO */
	timerdev.led_gpio = of_get_named_gpio(timerdev.nd, "led-gpios", 0);
	if (timerdev.led_gpio < 0)
	{
		goto fail_findnd;
	}

	/* 申请IO */
	ret = gpio_request(timerdev.led_gpio, "led_gpio");
	if (ret < 0)
	{
		printk("failed to request the led\r\n");
		/* NOTE:申请失败的话，大部分原因这个IO被别的外设占用
		 * 需要在设备树中屏蔽相关代码，或者status属性值设置为disable
		 * 检查复用，也就是pinctl
		 * gpio使用 
		 */
		goto fail_findnd;
	}

	/* 使用IO 设置为输出 默认为低电平*/
	ret = gpio_direction_output(timerdev.led_gpio, 0);
	if (ret < 0)
	{
		goto fail_setoutput;
	}

	init_timer(&timerdev.timer);	// 初始化定时器
	timerdev.timer.function = timer_function;
	timerdev.timer.expires = jiffies + msecs_to_jiffies(500);		// 两秒
	add_timer(&timerdev.timer);		// 添加到系统

	return 0;

fail_setoutput:
	gpio_free(timerdev.led_gpio);
fail_findnd:
	device_destroy(timerdev.class, timerdev.devid);
fail_device:
	class_destroy(timerdev.class);
fail_class:
	cdev_del(&timerdev.cdev);
fail_cdev:
	unregister_chrdev_region(timerdev.devid, TIMER_CNT);
fail_devid:
	return ret;
}

static void __exit timer_exit(void)
{
	/* 释放IO */
	gpio_free(timerdev.led_gpio);
	device_destroy(timerdev.class, timerdev.devid);
	class_destroy(timerdev.class);
	cdev_del(&timerdev.cdev);
	unregister_chrdev_region(timerdev.devid,TIMER_CNT);
	del_timer(&timerdev.timer);		// 删除定时器
	gpio_set_value(timerdev.led_gpio, 1);
}

/* 注册驱动和卸载驱动 */
module_init(timer_init);
module_exit(timer_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("fengyuhang");

```

# 使用`ioctl`函数控制

用户`APP`中使用`ioctl`函数

```c
unlocked_ioctl();
compat_ioctl();
```

对应驱动里的函数

```c
/*
 * 如果是64位的应用程序运行在64位的内核上，调用的是unlocked_ioctl，
 * 如果是32位的应用程序运行在32位的内核上，调用的也是unlocked_ioctl。
 */
long (*unlocked_ioctl)(struct file *, unsigned int, unsigned long);

/* 
 * 支持64位系统的驱动必须要实现的ioctl，当有32位的应用程序调用64位内核的ioctl时，
 * 这个回调函数会被调用，如果没有实现compat_ioctl，那么32位的应用程序在64位的内核上
 * 执行ioctl时会返回错误：Not a typewriter
 */
long (*compat_ioctl)(struct file *, unsigned int, unsigned long);
```

`ioctl`的命令是自己定义的，但是要符合`Linux`的**规则**。具体的说明可以查看博主[coolwriter](https://blog.csdn.net/coolwriter)的博客。

`cmd`拆分如下

| 幻数(type) | 序数(number) | 数据传输方向(direction) | 数据大小(size) |
| :--------: | :----------: | :---------------------: | :------------: |
|    8bit    |     8bit     |          2bit           |     14bit      |

- 幻数：是一个`0~0xFF`的数，是用来区分不同的书驱动的。
- 序数：用这个数来给自己的命令编号。
- 数据传输方向：如果涉及到要传参，内核要求描述一下传输的方向。
  - `_IOC_NONE`：值为0，无数据传输。
  - `_IOC_READ`：值为1，从设备驱动读取数据。
  - `_IOC_WRITE`：值为2，向设备驱动写入数据。
  - `_IOC_READ|_IOC_WRITE`：双向数据传输。
- 数据大小：==与体系结构相关==，`ARM`下占`14bit(_IOC_SIZEBITS)`，如果数据是`int`，内核给这个赋的值就是`sizeof(int)`。

构建命令

```c
/* type是幻数，nr是序数，size是大小 */
#define _IO(type, nr)			// 没有参数的命令
#define _IOR(type, nr, size)	// 该命令是从驱动读取数据
#define _IOW(type, nr, size)	// 该命令是从驱动写入数据
#define _IOWR(type, nr, size)	// 双向数据传输
```

驱动程序

```c
#include <linux/types.h>
#include <linux/kernel.h>
#include <linux/delay.h>
#include <linux/ide.h>
#include <linux/init.h>
#include <linux/module.h>
#include <linux/errno.h>
#include <linux/gpio.h>
#include <linux/cdev.h>
#include <linux/of.h>
#include <linux/of_address.h>
#include <linux/of_irq.h>
#include <linux/of_gpio.h>
#include <linux/timer.h>
#include <linux/jiffies.h>
#include <asm/mach/map.h>
#include <asm/uaccess.h>
#include <asm/io.h>


#define TIMER_CNT	1
#define TIMER_NAME	"timer"

/* 
 * 构建命令
 * 0xEF是在Linux内核文档中找的没用被其他驱动程序占用的。
 */
#define CLOSE_CMD				_IO(0xEF, 1)		// 关闭命令
#define OPEN_CMD				_IO(0xEF, 2)		// 打开命令
#define SET_PERIOD_CMD			_IOW(0xEF, 3, int)	// 设置周期

/* 定义设备驱动结构体 */
struct timer_dev
{
	dev_t devid;
	int major;
	int minor;
	struct cdev cdev;
	struct class *class;
	struct device *device;
	struct device_node *nd;
	int led_gpio;				/* led 所使用的 GPIO 编号 */
	struct timer_list timer;	/* 定义一个定时器 */
	atomic_t timePeriod;		/* 定时器时间周期，使用原子变量，保护数据 */
};

struct timer_dev timerdev;		/* 定义一个设备 */

static int timer_open(struct inode *inode, struct file *filp)
{
	filp->private_data = &timerdev;
	
	return 0;
}	

static int timer_release(struct inode *inode, struct file *filp)
{
	struct timer_dev *dev = filp->private_data;
	return 0;
}

static long timer_ioctl(struct file* filp, unsigned int cmd, unsigned long arg)
{
	struct timer_dev *dev = filp->private_data;
	int ret = 0;
	static int value = 0;
	switch (cmd)
	{
		case CLOSE_CMD:
			/* 关闭定时器 */
			del_timer_sync(&dev->timer);
			break;
		case OPEN_CMD:
			/* 设置时间点 */
			mod_timer(&dev->timer, jiffies + msecs_to_jiffies(atomic_read(&dev->timePeriod)));
			break;
		case SET_PERIOD_CMD:
			/* 根据APP程序设置的周期来设置定时器的时间点 */
			ret = copy_from_user(&value, (int *)arg, sizeof(int));
			if (ret < 0)
			{
				return -EFAULT;
			}
			atomic_set(&dev->timePeriod, value);
			mod_timer(&dev->timer, jiffies + msecs_to_jiffies(atomic_read(&dev->timePeriod)));
			break;
		default:
			break;
	}
	return 0;
}

static const struct file_operations timer_fops = {
	.owner = THIS_MODULE,
	.open = timer_open,
	.release = timer_release,
	.unlocked_ioctl = timer_ioctl,
};

/* 定时器处理函数 */
static void timer_function(unsigned long arg)
{
	static int i = 0;
    /* 修改时间点，并重新开启定时器。 */
	mod_timer(&timerdev.timer, jiffies + msecs_to_jiffies(atomic_read(&timerdev.timePeriod)));
	gpio_set_value(timerdev.led_gpio, i);
	i = !i;
}

/* 入口和出口 */
static int __init timer_init(void)
{
	int ret = 0;
	
	/* 注册字符设备ID */
	timerdev.major = 0;
	if(timerdev.major)
	{
		/* 给定设备号 */
		timerdev.devid = MKDEV(timerdev.major, 0);
		ret = register_chrdev_region(timerdev.devid, TIMER_CNT, TIMER_NAME);
	}
	else
	{
		alloc_chrdev_region(&timerdev.devid, 0,TIMER_CNT,TIMER_NAME);
	}

	if (ret < 0)
	{
		goto fail_devid;
	}

	/* 初始化字符设备 */
	timerdev.cdev.owner = THIS_MODULE;
	cdev_init(&timerdev.cdev, &timer_fops);
	ret = cdev_add(&timerdev.cdev, timerdev.devid, TIMER_CNT);
	if (ret < 0)
	{
		goto fail_cdev;
	}

	/* 创建类 */
	timerdev.class = class_create(THIS_MODULE, TIMER_NAME);
	if (IS_ERR(timerdev.class))
	{
		ret = PTR_ERR(timerdev.class);
		goto fail_class;
	}

	/* 创建设备 */
	timerdev.device = device_create(timerdev.class, NULL, timerdev.devid, NULL, TIMER_NAME);
	if (IS_ERR(timerdev.device))
	{
		ret = PTR_ERR(timerdev.device);
		goto fail_device;
	}

	/* 获取设备节点 */
	timerdev.nd = of_find_node_by_path("/gpioled");
	if (timerdev.nd == NULL)
	{
		ret = -EINVAL;
		goto fail_findnd;
	}

	/* 获取LED对应的GPIO */
	timerdev.led_gpio = of_get_named_gpio(timerdev.nd, "led-gpios", 0);
	if (timerdev.led_gpio < 0)
	{
		goto fail_findnd;
	}

	/* 申请IO */
	ret = gpio_request(timerdev.led_gpio, "led_gpio");
	if (ret < 0)
	{
		printk("failed to request the led\r\n");
		/* NOTE:申请失败的话，大部分原因这个IO被别的外设占用
		 * 需要在设备树中屏蔽相关代码，或者status属性值设置为disable
		 * 检查复用，也就是pinctl
		 * gpio使用 
		 */
		goto fail_findnd;
	}

	/* 使用IO 设置为输出 默认为低电平*/
	ret = gpio_direction_output(timerdev.led_gpio, 0);
	if (ret < 0)
	{
		goto fail_setoutput;
	}

	init_timer(&timerdev.timer);				// 初始化定时器
	atomic_set(&timerdev.timePeriod, 500);		// 设置初始周期值
	timerdev.timer.function = timer_function;
	timerdev.timer.expires = jiffies + msecs_to_jiffies(atomic_read(&timerdev.timePeriod));		// 500ms
	add_timer(&timerdev.timer);					// 添加到系统

	return 0;

fail_setoutput:
	gpio_free(timerdev.led_gpio);
fail_findnd:
	device_destroy(timerdev.class, timerdev.devid);
fail_device:
	class_destroy(timerdev.class);
fail_class:
	cdev_del(&timerdev.cdev);
fail_cdev:
	unregister_chrdev_region(timerdev.devid, TIMER_CNT);
fail_devid:
	return ret;
}

static void __exit timer_exit(void)
{
	/* 释放IO */
	gpio_free(timerdev.led_gpio);
	device_destroy(timerdev.class, timerdev.devid);
	class_destroy(timerdev.class);
	cdev_del(&timerdev.cdev);
	unregister_chrdev_region(timerdev.devid,TIMER_CNT);
	del_timer(&timerdev.timer);		// 删除定时器
	gpio_set_value(timerdev.led_gpio, 1);
}

/* 注册驱动和卸载驱动 */
module_init(timer_init);
module_exit(timer_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("fengyuhang");

```

测试APP程序

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/ioctl.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

/* 驱动里定义的命令 */
#define CLOSE_CMD				_IO(0xEF, 1)		// 关闭命令
#define OPEN_CMD				_IO(0xEF, 2)		// 打开命令
#define SET_PERIOD_CMD			_IOW(0xEF, 3, int)	// 设置周期

/**
 * ./timerAPP <filename> <0:1>
 * @param argc 应用程序参数个数
 * @param argv 保存的参数，字符串形式。
 * */
int main(int argc, char *argv[])
{
	int ret = 0;
	int fd = 0;
	char *filename;
	unsigned int cmd;
	unsigned int arg;

/*
	if (argc != 3)
	{
		printf("输入错误\r\n");
	}
*/

	filename = argv[1];

	fd = open(filename, O_RDWR);
	if(fd < 0) {
		printf("Error: Open %s fail.\r\n",filename);
		return -1;
	}

	while (1)
	{
		printf("请输入命令(1:关闭定时器，2:打开定时器，3:设置周期，4:关闭文件):");
		ret = scanf("%d", &cmd);
		fflush(stdin);				// 清空输入缓存，防止下次scanf时读取到上次遗留下的回车\n。
		if (cmd == 1)
		{
			ioctl(fd, CLOSE_CMD, 0);
		}else if (cmd == 2)
		{
			ioctl(fd, OPEN_CMD, 0);
		}else if (cmd == 3)
		{
			printf("请输入定时器周期：");
			ret = scanf("%d", &arg);
			fflush(stdin);
			ioctl(fd, SET_PERIOD_CMD, &arg);
		}
		else if (cmd == 4)
		{
			break;
		}
	}
	
	/* 关闭 */
	ret = close(fd);
	if (ret < 0)
	{
		printf("Error: Close %s fail.\r\n", filename);
	}
	
	return 0;
}
```

