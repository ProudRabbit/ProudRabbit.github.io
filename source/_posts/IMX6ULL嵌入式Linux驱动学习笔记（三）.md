---
title: IMX6ULL嵌入式Linux驱动学习笔记（三）
date: 2020-09-19 21:44:12
tags: [嵌入式, IMX6ULL, 学习笔记]
keywords:
description: 正点原子alpha开发板IMX6ULL嵌入式Linux开发学习笔记。
---

**IMX6ULL嵌入式Linux驱动开发学习**

以下内容是我在学习正点原子`IMX6ULL`开发板`alpha`中记录的笔记，部分摘录自正点原子`IMX6ULL开发手册`。

## 一、地址映射

1. 因为`linux`使用`MMC`，因此在驱动开发时，不能直接对寄存器物理地址进行读写操作。

2. 在`linux`里面操作的都是虚拟地址，所以需要先得到物理地址对应的虚拟地址。获得物理地址对应的虚拟地址使用`va = ioremap(cookie,size)`函数，第一个参数是物理地址起始地址，第二个参数就是要转换的字节数量，返回的是申请到的虚拟地址。卸载驱动的时候使用`iounmap(va)`；

   <!--more-->

3. 操作虚拟地址时使用

   - `readb(const volatile void __iomem *addr)`   8bit
   - `readw(const volatile void __iomem *addr)`   16bit
   - `readl(const volatile void __iomem *addr)`   32bit
   - `writeb(u8 value,volatile void __iomem *addr)`    8bit
   - `writew(u16 value, volatile void __iomem *addr)`    16bit
   - `writel(u32 value, volatile void __iomem *addr)`    32bit

```c
/**
 * @brief 出口
 * */
static int __init led_init(void)
{
	int ret = 0;
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

	ret = register_chrdev(LED_MAJOR, LED_NAME, &led_fops);
	if (ret < 0)
	{
		printk("led_init failed! \r\n");
		return -EIO;
	}

	printk("led_init\r\n");
	return 0;
}

/**
 * @brief 出口
 * */
static void __exit led_exit(void)
{
	/* 取消地址映射 */
	iounmap(CCM_CCGR1);
	iounmap(SW_MUX_GPIO1_IO03);
	iounmap(SW_PAD_GPIO1_IO03);
	iounmap(GPIO1_GDIR);
	iounmap(GPIO1_DR);

	/* 注销字符设备 */
	unregister_chrdev(LED_MAJOR,LED_NAME);
	printk("led_exit\r\n");
}
```

## 二、驱动程序编写（正常开发中不使用这种方式）

1. 初始化时钟、IO、GPIO等。  
2. **如果要在卸载驱动时关闭LED，一定要在取消地址映射前操作LED。**  

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

#define LED_MAJOR	100		// 主设备号
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

static int led_open(struct inode *inode, struct file *filp)
{

	return 0;
}

static int led_release(struct inode *inode, struct file *filp)
{
	return 0;
}

static ssize_t led_write(struct file * fp, const char __user *buf, size_t len, loff_t * off)
{
	int retvalue;
	unsigned char databuff[1];
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

/* 字符设备操作集 */
static const struct file_operations led_fops = {
	.owner = THIS_MODULE,
	.write = led_write,
	.open = led_open,
	.release = led_release,
};

/**
 * @brief 模块入口函数
 * */
static int __init led_init(void)
{
	int ret = 0;
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

	ret = register_chrdev(LED_MAJOR, LED_NAME, &led_fops);
	if (ret < 0)
	{
		printk("led_init failed! \r\n");
		return -EIO;
	}

	printk("led_init\r\n");
	return 0;
}

/**
 * @brief 出口
 * */
static void __exit led_exit(void)
{
    /* 如果要在卸载驱动时关闭LED，一定要在取消地址映射前操作LED */
	/* 取消地址映射 */
	iounmap(CCM_CCGR1);
	iounmap(SW_MUX_GPIO1_IO03);
	iounmap(SW_PAD_GPIO1_IO03);
	iounmap(GPIO1_GDIR);
	iounmap(GPIO1_DR);

	/* 注销字符设备 */
	unregister_chrdev(LED_MAJOR,LED_NAME);
	printk("led_exit\r\n");
}

/**
 * 驱动的加载和卸载
 * */
module_init(led_init);
module_exit(led_exit);

MODULE_LICENSE("GPL");
```

## 三、应用程序编写  

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
/* 	if (atoi(argv[2]) ==1 )		// 传递过来的是字符串，需要转换成数字
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

## 四、测试

1. 先输入 `depmod`。
2. 然后输入 `modprobe led.ko` 加载驱动
3. 再输入 `mknod /dev/led` 创建设备节点
4. 输入 `./ledAPP /dev/led 0` 或 `./ledAPP /dev/led 1` 来点亮和关闭 `led` 。