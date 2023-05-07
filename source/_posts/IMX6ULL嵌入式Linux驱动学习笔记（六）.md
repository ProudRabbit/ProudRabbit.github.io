---
title: IMX6ULL嵌入式Linux驱动学习笔记（六）
date: 2020-09-19 21:55:47
tags: [嵌入式, IMX6ULL, 学习笔记]
keywords:
description: 正点原子alpha开发板IMX6ULL嵌入式Linux开发学习笔记。
---

**IMX6ULL嵌入式Linux驱动开发学习**

以下内容是我在学习正点原子`IMX6ULL`开发板`alpha`中记录的笔记，部分摘录自正点原子`IMX6ULL开发手册`。

**正常工作中进行驱动开发的方式——子系统。**

## 一、pinctrl子系统

​	借助`pinctrl`来设置一个`pin`的复用和电气属性。  

​	`pinctrl` 子系统主要工作内容如下：

1. 获取设备树中的`pin`信息。  
2. 根据获取到的`pin`信息来设置`pin`的复用功能。  
3. 根据获取到的`pin`信息来设置`pin`的电气特性，比如上/下拉、速度、驱动能力等。   

<!--more-->

对于使用者来说，只要在设备树里面设置某个`pin`的相关属性即可，其他的初始化工作由`pinctrl`子系统来完成，`pinctrl`子系统源码目录为`drivers/pinctrl`。根据设备的类型，会创建对应的子节点，然后设备所用`pin`都放到此节点。

```dtd
imx6ul-evk {
	pinctrl_hog_1: hoggrp-1 {
		fsl,pins = <
			MX6UL_PAD_UART1_RTS_B__GPIO1_IO19		0x17059 /* SD1 CD */
			MX6UL_PAD_GPIO1_IO05__USDHC1_VSELECT	0x17059 /* SD1 VSELECT */
			MX6UL_PAD_GPIO1_IO09__GPIO1_IO09        0x17059 /* SD1 RESET */
		>;
	};
	......
};
```

## 二、gpio子系统

​	使用`gpio`子系统来使用`gpio`。  

## 三、驱动编写  

1. 设备树修改  

```c
/ {
    ......
        
  	gpioled {
		compatible = "atkalpha,gpioled";
		pinctrl-names = "default";
		pinctrl-0 = <&pinctrl_gpioled>;
		led-gpios = <&gpio1 3 GPIO_ACTIVE_LOW>;
		status = "okay";
		
	};  
};

&iomuxc {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_hog_1>;
	imx6ul-evk {
		pinctrl_hog_1: hoggrp-1 {
			fsl,pins = <
				MX6UL_PAD_UART1_RTS_B__GPIO1_IO19	0x17059 /* SD1 CD */
				MX6UL_PAD_GPIO1_IO05__USDHC1_VSELECT	0x17059 /* SD1 VSELECT */
				MX6UL_PAD_GPIO1_IO09__GPIO1_IO09        0x17059 /* SD1 RESET */
			>;
		};

		/* 自己定义的led */
		pinctrl_gpioled: ledgrp {
			fsl,pins = <
				MX6UL_PAD_GPIO1_IO03__GPIO1_IO03	0X10B0
			>;
		};
        ......
    };
};

&tsc {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_tsc>;
	xnur-gpio = <&gpio1 3 GPIO_ACTIVE_LOW>;
	measure-delay-time = <0xffff>;
	pre-charge-time = <0xfff>;
	status = "disable";	// 因为和LED灯使用引脚冲突，所以需要关闭
};
```

2. 驱动程序  

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
#include <asm/mach/map.h>
#include <asm/uaccess.h>
#include <asm/io.h>


#define GPIOLED_CNT		1
#define GPIOLED_NAME	"gpioled"

#define LEDOFF	0
#define LEDON 	1

struct gpioled_dev
{
	dev_t devid;
	int major;
	int minor;
	struct cdev cdev;
	struct class *class;
	struct device *device;
	struct device_node *nd;
	int led_gpio;			/* led 所使用的 GPIO 编号 */
};

struct gpioled_dev gpioled;

static int led_open(struct inode *inode, struct file *filp)
{
	filp->private_data = &gpioled;
	
	return 0;
}	

static int led_release(struct inode *inode, struct file *filp)
{
	struct gpioled_dev *dev = filp->private_data;
	return 0;
}

ssize_t led_write(struct file *filp, const char __user *buf, size_t count, loff_t *ppos)
{
	struct gpioled_dev *dev = filp->private_data;
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

/* 入口和出口 */
static int __init led_init(void)
{
	int ret = 0;
	
	/* 注册字符设备驱动 */
	gpioled.major = 0;
	if(gpioled.major)
	{
		/* 给定设备号 */
		gpioled.devid = MKDEV(gpioled.major, 0);
		ret = register_chrdev_region(gpioled.devid, GPIOLED_CNT, GPIOLED_NAME);
	}
	else
	{
		alloc_chrdev_region(&gpioled.devid, 0,GPIOLED_CNT,GPIOLED_NAME);
	}

	if (ret < 0)
	{
		goto fail_devid;
	}

	/* 初始化cdev */
	gpioled.cdev.owner = THIS_MODULE;
	cdev_init(&gpioled.cdev, &led_fops);
	ret = cdev_add(&gpioled.cdev, gpioled.devid, GPIOLED_CNT);
	if (ret < 0)
	{
		goto fail_cdev;
	}

	/* 创建类 */
	gpioled.class = class_create(THIS_MODULE, GPIOLED_NAME);
	if (IS_ERR(gpioled.class))
	{
		ret = PTR_ERR(gpioled.class);
		goto fail_class;
	}
	
    /* 创建设备节点 */
	gpioled.device = device_create(gpioled.class, NULL, gpioled.devid, NULL, GPIOLED_NAME);
	if (IS_ERR(gpioled.device))
	{
		ret = PTR_ERR(gpioled.device);
		goto fail_device;
	}

	/* 获取设备节点 */
	gpioled.nd = of_find_node_by_path("/gpioled");
	if (gpioled.nd == NULL)
	{
		ret = -EINVAL;
		goto fail_findnd;
	}

	/* 获取LED对应的GPIO */
	gpioled.led_gpio = of_get_named_gpio(gpioled.nd, "led-gpios", 0);
	if (gpioled.led_gpio < 0)
	{
		goto fail_findnd;
	}

	/* 申请IO */
	ret = gpio_request(gpioled.led_gpio, "led_gpio");
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
	ret = gpio_direction_output(gpioled.led_gpio, 0);
	if (ret < 0)
	{
		goto fail_setoutput;
	}

	return 0;

fail_setoutput:
	gpio_free(gpioled.led_gpio);
fail_findnd:
	device_destroy(gpioled.class, gpioled.devid);
fail_device:
	class_destroy(gpioled.class);
fail_class:
	cdev_del(&gpioled.cdev);
fail_cdev:
	unregister_chrdev_region(gpioled.devid, GPIOLED_CNT);
fail_devid:
	return ret;
}

static void __exit led_exit(void)
{
	/* 释放IO */
	gpio_free(gpioled.led_gpio);
	device_destroy(gpioled.class, gpioled.devid);
	class_destroy(gpioled.class);
	cdev_del(&gpioled.cdev);
	unregister_chrdev_region(gpioled.devid,GPIOLED_CNT);
}

/* 注册驱动和卸载驱动 */
module_init(led_init);
module_exit(led_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("fengyuhang");
```



