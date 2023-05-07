---
title: IMX6ULL嵌入式Linux驱动学习笔记（十一）
date: 2021-01-11 14:26:52
tags: [嵌入式, IMX6ULL, 学习笔记]
keywords:
description: 正点原子alpha开发板IMX6ULL嵌入式Linux开发学习笔记。
---

**IMX6ULL嵌入式Linux驱动开发学习**

以下内容是我在学习正点原子`IMX6ULL`开发板`alpha`中记录的笔记，部分摘录自正点原子`IMX6ULL开发手册`。

## 一、异步通知简介

### 1.1 硬件中断

中断是处理器提供的一种异步机制，配置好中断以后就可以让处理器去处理其他的事情了，当中断发生以后会触发事先设置好的中断服务函数，在中断服务函数中做具体的处理。

### 1.2 信号

信号类似于我们硬件上使用的“中断”，信号是软件层次上对中断机制的一种模拟，也叫作软中断信号。**和软中断不是一个概念。**

<!--more-->

驱动可以通过主动向应用程序发送信号的方式来报告自己可以访问了，应用程序获取到信号以后就可以从驱动设备中读取或者写入数据。整个过程就相当于应用程序收到了驱动发送过来了的一个中断，然后应用程序去响应这个中断。

异步通知的核心就是信号，在`arch/xtensa/include/uapi/asm/signal.h`中定义了`Linux`所持的信号，如下所示。

```c
#define SIGHUP 		1 	/* 终端挂起或控制进程终止 */
#define SIGINT 		2 	/* 终端中断(Ctrl+C 组合键) */
#define SIGQUIT 	3 	/* 终端退出(Ctrl+\组合键) */
#define SIGILL 		4 	/* 非法指令 */
#define SIGTRAP 	5 	/* debug 使用，有断点指令产生 */
#define SIGABRT 	6 	/* 由 abort(3)发出的退出指令 */
#define SIGIOT 		6 	/* IOT 指令 */
#define SIGBUS 		7 	/* 总线错误 */
#define SIGFPE 		8 	/* 浮点运算错误 */
#define SIGKILL 	9 	/* 杀死、终止进程 */
#define SIGUSR1 	10 	/* 用户自定义信号 1 */
#define SIGSEGV 	11 	/* 段违例(无效的内存段) */
#define SIGUSR2 	12 	/* 用户自定义信号 2 */
#define SIGPIPE 	13 	/* 向非读管道写入数据 */
#define SIGALRM 	14 	/* 闹钟 */
#define SIGTERM 	15 	/* 软件终止 */
#define SIGSTKFLT 	16 	/* 栈异常 */
#define SIGCHLD 	17 	/* 子进程结束 */
#define SIGCONT 	18 	/* 进程继续 */
#define SIGSTOP 	19 	/* 停止进程的执行，只是暂停 */
#define SIGTSTP 	20 	/* 停止进程的运行(Ctrl+Z 组合键) */
#define SIGTTIN 	21 	/* 后台进程需要从终端读取数据 */
#define SIGTTOU 	22 	/* 后台进程需要向终端写数据 */
#define SIGURG 		23 	/* 有"紧急"数据 */
#define SIGXCPU 	24 	/* 超过 CPU 资源限制 */
#define SIGXFSZ 	25 	/* 文件大小超额 */
#define SIGVTALRM 	26 	/* 虚拟时钟信号 */
#define SIGPROF 	27 	/* 时钟信号描述 */
#define SIGWINCH 	28 	/* 窗口大小改变 */
#define SIGIO 		29 	/* 可以进行输入/输出操作 */
#define SIGPOLL 	SIGIO
/* #define SIGLOS 29 */
#define SIGPWR 		30 	/* 断点重启 */
#define SIGSYS 		31 	/* 非法的系统调用 */
#define SIGUNUSED 	31 	/* 未使用信号 */
```

这些信号中，除了 `SIGKILL(9)`和 `SIGSTOP(19)`这两个信号不能被忽略外，其他的信号都可以忽略。这些信号就相当于中断号，不同的中断号代表了不同的中断，不同的中断所做的处理不同，因此，驱动程序可以通过向应用程序发送不同的信号来实现不同的功能。

### 1.3 信号处理函数

使用中断的时候需要设置中断处理函数，同样的，如果要在应用程序中使用信号，那么就必须设置信号所使用的信号处理函数，在**应用程序**中使用 `signal` 函数来设置指定信号的处理函数， `signal` 函数原型如下所示

```c
/**
 * @param signum 要设置处理函数的信号。
 * @param handler 信号的处理函数。
 * @return 设置成功的话返回信号的前一个处理函数，设置失败的话返回 SIG_ERR。
 */
sighandler_t signal(int signum, sighandler_t handler);

/* 信号处理函数原型 */
typedef void (*sighandler_t)(int);
```

### 1.4 驱动中对异步通知的处理

1. 需要在驱动程序中定义一个`fasync_struct`结构体指针变量，`fasync_struct`结构体内容如下：

   >```c
   >struct fasync_struct {
   >spinlock_t fa_lock;
   >int magic;
   >int fa_fd;
   >struct fasync_struct *fa_next;
   >struct file *fa_file;
   >struct rcu_head fa_rcu;
   >};
   >```
   >
   >一般将`fasync_struct`结构体指针变量定义到设备结构体中。
   >
   >```c
   >struct imx6uirq_dev {
   >struct device *dev;
   >struct class *cls;
   >struct cdev cdev;
   >
   >	......
   >
   >	struct fasync_struct *async_queue;		/* 异步相关结构体 */
   >};
   >```

2. 要实现`file_operations`里面的`fasync`函数。

   > 要使用异步通知，需要在设备驱动中实现 `file_operations` 操作集中的 `fasync` 函数，格式如下所示
   >
   > ```c
   > int (*fasync) (int fd, struct file *filp, int on);
   > ```
   >
   > `fasync` 函数里面一般通过调用 `fasync_helper` 函数来初始化前面定义的 `fasync_struct `结构体指针， `fasync_helper` 函数原型如下：
   >
   > ```c
   > int fasync_helper(int fd, struct file * filp, int on, struct fasync_struct **fapp);
   > ```
   >
   > 当应用程序通过 `fcntl(fd, F_SETFL, flags | FASYNC)` 改变 `fasync` 标记的时候，驱动程序 `file_operations` 操作集中的 `fasync` 函数就会执行。

3. 向应用程序发送信号，`kill_fasync`函数

   >当设备可以访问的时候，驱动程序需要向应用程序发出信号，相当于产生“中断”。`kill_fasync` 函数负责发送指定的信号， `kill_fasync` 函数原型如下所示
   >
   >```c
   >/**
   >* @param fp 要操作的 fasync_struct。
   >* @param sig 要发送的信号
   >* @param band 可读时设置为 POLL_IN，可写时设置为 POLL_OUT。
   >*/
   >void kill_fasync(struct fasync_struct **fp, int sig, int band);
   >```

4. 关闭驱动时，需要删除信号。

   > 关闭驱动文件的时候需要在 `file_operations` 操作集中的 `release` 函数中释放 `fasync_struct`，`fasync_struct` 的释放函数同样为 `fasync_helper`。
   >
   > ```c
   > static int xxx_release(struct inode *inode, struct file *filp)
   > {
   > 	return xxx_fasync(-1, filp, 0);		/* 删除异步通知 */
   > }
   > ```

 **驱动中 `fasync` 函数参考示例**

```c
struct xxx_dev {
	......
        
	struct fasync_struct *async_queue;	/* 异步相关结构体 */
};

static int xxx_release(struct inode *inode, struct file *filp)
{
    return xxx_fasync(-1, filp, 0);		/* 删除异步通知 */
}

static int xxx_fasync(int fd, struct file *filp, int on)
{
    struct xxx_dev *dev = (xxx_dev)filp->private_data;
    
    if (fasync_helper(fd, filp, on, &dev->async_queue) < 0)
    	return -EIO;
    return 0;
}

static struct file_operations xxx_ops = {
    ......
        
    .fasync = xxx_fasync,
    
    ......
};
```

### 1.5 应用程序对异步通知的处理

1. 注册信号处理函数

   >使用 `signal()` 函数来设置指定信号的处理函数。
   >
   >```c
   >/* 设置信号处理函数 */
   >signal(SIGIO, sigio_signal_func);
   >```

2. 将本应用程序的进程号告诉给内核

   使用 `fcntl(fd, F_SETOWN, getpid());` 将本应用程序的进程号告诉给内核。 

3. 开启异步通知

   > 使用如下两行程序开启异步通知
   >
   > ```c
   > /* 开启异步通知 */
   > flags = fcntl(fd, F_GETFL);				/* 获取当前的进程状态 */
   > fcntl(fd, F_SETFL, flags | FASYNC);		/* 开启当前进程异步通知功能 */
   > ```
   >
   > **重点就是通过 `fcntl` 函数设置进程状态为 `FASYNC`，经过这一步，驱动程序中的 `fasync `函数就会执行。**

## 二、实验驱动编写

### 2.1 驱动程序

```c
#include <linux/types.h>
#include <linux/kernel.h>
#include <linux/delay.h>
#include <linux/init.h>
#include <linux/ide.h>
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
#include <linux/string.h>
#include <linux/irq.h>
#include <linux/interrupt.h>
#include <asm/mach/map.h>
#include <asm/uaccess.h>
#include <asm/io.h>


#define KEYIRQ_CNT			1
#define KEYIRQ_NAME			"keyirq"
#define KEY_NUM				1
#define KEY0_VALUE			0X01
#define INVAKEY				0XFF	/* 无效按键值 */

struct key_dev
{
	int gpio;								/* IO编号 */
	int irqNum;								/* 中断号 */
	unsigned char value;					/* 按键值 */
	char name[10];							/* 名称 */
	irqreturn_t (*handler)(int, void*);		/* 中断处理函数 */

	struct tasklet_struct tasklet;
};


/* 定义设备驱动结构体 */
struct keyirq_dev
{
	dev_t devid;						/* 设备ID */
	int major;							/* 主设备号 */
	int minor;							/* 次设备号 */
	struct cdev cdev;					/* 字符设备结构体 */
	struct class *class;				/* 设备类 */
	struct device *device;				/* 设备节点 */
	struct device_node *nd;				/* 设备树节点 */
	struct key_dev key[KEY_NUM];
	struct timer_list timer;			/* 定时器 */

	atomic_t keyValue;					/* 按键值 */
	atomic_t keyRelease;				/* 按键是否释放 */

	struct fasync_struct *fasync;		/* 定义一个异步通知结构体 */
	
};

struct keyirq_dev keyirq;		/* 定义一个keyirq设备 */


static int keyirq_fasync(int fd, struct file *filp, int on)
{
	struct keyirq_dev *dev = filp->private_data;

	return fasync_helper(fd, filp, on, &dev->fasync);
}


static int keyirq_open(struct inode *inode, struct file *filp)
{
	filp->private_data = &keyirq;
	
	return 0;
}	

static int keyirq_release(struct inode *inode, struct file *filp)
{
	struct keyirq_dev *dev = filp->private_data;

	return keyirq_fasync(-1, filp, 0);
	
}

static ssize_t keyirq_write(struct file *filp, const char __user *buf, size_t count, loff_t *ppos)
{
	struct keyirq_dev *dev = filp->private_data;
	
	return 0;
}

static ssize_t keyirq_read(struct file *filp, char __user *buf, size_t count, loff_t *ppos)
{
	struct keyirq_dev *dev = filp->private_data;
	int ret = 0;
	unsigned char keyValue;
	unsigned char keyRelease;

	keyValue = atomic_read(&dev->keyValue);
	keyRelease = atomic_read(&dev->keyRelease);

	if (keyRelease)
	{
		/* 一次有效按键 */
		if (keyValue & 0X80)
		{
			keyValue &= ~0X80;			// 去除标记
			ret = copy_to_user(buf, &keyValue, sizeof(keyValue));
		}
		else
		{
			goto data_err;
		}
		atomic_set(&dev->keyRelease, 0);
	} 
	else
	{
		goto data_err;
	}
	

	return ret;
data_err:
	return -EINVAL;

}


/* 操作集 */
struct file_operations keyirq_fops = {
	.owner 		= THIS_MODULE,
	.open 		= keyirq_open,
	.release 	= keyirq_release,
	.write 		= keyirq_write,
	.read 		= keyirq_read,
	.fasync 	= keyirq_fasync, 
};

/**
 * @brief 定时器处理函数
 * @param arg 用户参数
 */
static void timer_func(unsigned long arg)
{
	struct keyirq_dev *dev = (struct keyirq_dev*)arg;
	int value = 0;

	value = gpio_get_value(dev->key[0].gpio);
	if (value == 0)
	{
		// 按键按下
		printk("KEY0 Press!\r\n");
		atomic_set(&dev->keyValue, dev->key[0].value);
	}
	else if(value == 1)
	{
		// 按键释放
		printk("KEY0 Release!\r\n");
		atomic_set(&dev->keyValue, dev->key[0].value | 0X80);		// 打上标签，标记按键按下
		atomic_set(&dev->keyRelease, 1);
	}

	if (atomic_read(&dev->keyRelease) == 1)
	{
		/* 如果是一次有效的按键 */
		kill_fasync(&dev->fasync, SIGIO, POLL_IN);
	}
}

/**
 * @brief key0 tasklet 处理函数
 * @param arg 传递的参数
 */ 
static void key0_tasklet(unsigned long arg)
{
	struct keyirq_dev *dev = (struct keyirq_dev *)arg;
	
	/* 启动定时器来延时消抖 */
	dev->timer.data = (volatile unsigned long)arg;
	mod_timer(&dev->timer, jiffies + msecs_to_jiffies(20));
}

/**
 * @brief 按键中断处理函数
 * @param irq 中断号
 */
static irqreturn_t key0_handler(int irq, void *arg)
{
	struct keyirq_dev *dev = arg;

	tasklet_schedule(&dev->key[0].tasklet);		/* 调度对应的tasklet */
	return IRQ_HANDLED;
}

/**
 * @brief 按键初始化
 * @param dev 设备结构体
 * @return 错误类型
 */
static int keyio_init(struct keyirq_dev *dev)
{
	int ret = 0;
	int i = 0;

	/* 按键初始化 */
	dev->nd = of_find_node_by_path("/key");
	if (dev->nd == NULL)
	{
		ret = -EINVAL;
		goto fail_nd;
	}

	/* 可能有多个按键，因此需要根据实际数量来读取 */
	for (i = 0; i < KEY_NUM; i++)
	{
		dev->key[i].gpio = of_get_named_gpio(dev->nd, "key-gpios", i);
		if (dev->key[i].gpio < 0 )
		{
			ret = -EINVAL;
			goto fail_getGpio;
		}
	}
	
	for (i = 0; i < KEY_NUM; i++)
	{
		memset(dev->key[i].name, 0, sizeof(dev->key[i].name));
		sprintf(dev->key[i].name, "KEY%d", i);
		ret = gpio_request(dev->key[i].gpio, dev->key[i].name);
		if (ret < 0 )
		{
			goto fail_requestGpio;
		}
		ret = gpio_direction_input(dev->key[i].gpio);
		if (ret < 0)
		{
			goto fail_setGpioDir;
		}

		/* 获取中断号，两种方式皆可 */
		dev->key[i].irqNum = gpio_to_irq(dev->key[i].gpio);
		// dev->key[i].irqNum = irq_of_parse_and_map(dev->nd, i);

	}
	
	/* 按键中断初始化 */
	dev->key[0].handler = key0_handler;
	dev->key[0].value = KEY0_VALUE;

	for (i = 0; i < KEY_NUM; i++)
	{
		/* 跳变沿触发方式 */
		ret = request_irq(dev->key[i].irqNum, dev->key[i].handler, IRQF_TRIGGER_RISING | IRQF_TRIGGER_FALLING, dev->key[i].name, dev);
		if (ret < 0)
		{
			printk("irq %d request failed!\r\n", dev->key[i].irqNum);
			goto fail_irq;
		}

		/* NOTE:没有根据多个按键来初始化各自的tasklet,实际中需要根据情况来编写 */
		tasklet_init(&dev->key[i].tasklet, key0_tasklet, (unsigned long)dev);
	}

	/* 初始化定时器 */
	init_timer(&dev->timer);
	dev->timer.function = timer_func;
	return 0;

fail_irq:
fail_setGpioDir:
	for (i = i-1; i >= 0; i--)
	{
		gpio_free(dev->key[i].gpio);
	}
fail_requestGpio:
fail_getGpio:
fail_nd:
	return ret;

}


/* 入口和出口 */
static int __init keyirq_init(void)
{
	int ret = 0;
	
	/* 注册字符设备ID */
	keyirq.major = 0;
	if(keyirq.major)
	{
		/* 给定设备号 */
		keyirq.devid = MKDEV(keyirq.major, 0);
		ret = register_chrdev_region(keyirq.devid, KEYIRQ_CNT, KEYIRQ_NAME);
	}
	else
	{
		alloc_chrdev_region(&keyirq.devid, 0,KEYIRQ_CNT,KEYIRQ_NAME);
		keyirq.major = MAJOR(keyirq.devid);
		keyirq.minor = MINOR(keyirq.devid);
		printk("dev Major ID:%d\r\n",keyirq.major);
	}

	if (ret < 0)
	{
		goto fail_devid;
	}

	/* 初始化字符设备 */
	keyirq.cdev.owner = THIS_MODULE;
	cdev_init(&keyirq.cdev, &keyirq_fops);
	ret = cdev_add(&keyirq.cdev, keyirq.devid, KEYIRQ_CNT);
	if (ret < 0)
	{
		goto fail_cdev;
	}

	/* 创建设备类 */
	keyirq.class = class_create(THIS_MODULE, KEYIRQ_NAME);
	if (IS_ERR(keyirq.class))
	{
		ret = PTR_ERR(keyirq.class);
		goto fail_class;
	}

	/* 创建设备节点 */
	keyirq.device = device_create(keyirq.class, NULL, keyirq.devid, NULL, KEYIRQ_NAME);
	if (IS_ERR(keyirq.device))
	{
		ret = PTR_ERR(keyirq.device);
		goto fail_device;
	}

	/* 初始化IO */
	ret = keyio_init(&keyirq);
	if (ret < 0)
	{
		goto fail_init;
	}

	/* 初始化原子变量 */
	atomic_set(&keyirq.keyValue, INVAKEY);
	atomic_set(&keyirq.keyRelease, 0);

	return 0;

fail_init:
	device_destroy(keyirq.class, keyirq.devid);
fail_device:
	class_destroy(keyirq.class);
fail_class:
	cdev_del(&keyirq.cdev);
fail_cdev:
	unregister_chrdev_region(keyirq.devid, KEYIRQ_CNT);
fail_devid:
	return ret;
}

static void __exit keyirq_exit(void)
{
	int i = 0;
	
	for (i = 0; i < KEY_NUM; i++)
	{
		/* 释放中断和IO */
		free_irq(keyirq.key[i].irqNum, &keyirq);
		gpio_free(keyirq.key[i].gpio);
	}

	/* 删除定时器 */
	del_timer_sync(&keyirq.timer);

	/* 注销字符设备驱动 */
	device_destroy(keyirq.class, keyirq.devid);
	class_destroy(keyirq.class);
	cdev_del(&keyirq.cdev);
	unregister_chrdev_region(keyirq.devid,KEYIRQ_CNT);

}

/* 注册驱动和卸载驱动 */
module_init(keyirq_init);
module_exit(keyirq_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("fengyuhang");
```

### 2.2 测试APP程序

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/ioctl.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <signal.h>

int fd;

/**
 * @brief 信号处理函数
 */
void sigio_signal_func(int num)
{
	int ret;
	unsigned char data;
	ret = read(fd, &data, sizeof(data));

	if (ret < 0)
	{
		/* code */
	}
	else
	{
		printf("sigio signal! key value = %d\r\n", data);
	}
}


/**
 * ./asyncnotiAPP /dev/keyirq
 * @param argc 应用程序参数个数
 * @param argv 保存的参数，字符串形式。
 * */
int main(int argc, char *argv[])
{
	int ret = 0;
	char *filename;
	unsigned char data;
	int flags = 0;

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

	/* 设置信号处理函数 */
	signal(SIGIO, sigio_signal_func);

	fcntl(fd, F_SETOWN, getpid());			/* 设置当前进程接收SIGIO信号 */
    
    /* 开启异步通知 */
	flags = fcntl(fd, F_GETFL);				/* 获取当前的进程状态 */
	fcntl(fd, F_SETFL, flags | FASYNC);		/* 开启当前进程异步通知功能 */

	while (1)
	{
		sleep(2);
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

