---
title: IMX6ULL学习笔记(二)
date: 2020-09-18 22:50:13
tags: [嵌入式, IMX6ULL, 学习笔记]
keywords:
description: 正点原子alpha开发板IMX6ULL裸机开发学习笔记。
---

**IMX6ULL裸机开发学习**

以下内容是我在学习正点原子`IMX6ULL`开发板`alpha`中记录的笔记，部分摘录自正点原子`IMX6ULL开发手册`。

## C语言运行环境构建

1. 设置处理器模式

   设置6ULL处于SVC模式下，设置CPSR寄存器的bit4-0，就是M[4:0]为10011=0X13，读写状态寄存器需要用MRS和MSR指令，

   

   <!--more-->

   

2. 设置sp指针  

   `sp`可以指向内部RAM，也可以指向DDR，我们将其指向DDR。512MB的范围 0X80000000～0X9FFFFFFF。栈大小设置为 0X200000=2MB。处理器栈增长模式，A7是向下增长的。设置SP=0X80200000   

3. 跳转到C语言  

   使用b指令，跳转到C语言函数，比如跳转到main函数。

代码如下：

```
.global _start	/* 全局标号 */

/*
 * 描述： _start函数，程序从此函数开始执行，主要是完成C语言运行环境设置
 *
 */
_start:
	@ 设置处理器进入SVC模式
	mrs r0, cpsr
	bic r0, r0, #0x1f		@ 将r0的低五位清零，也就是M[4:0]
	orr r0, r0, #0x13		@ r0或上0x13 表示使用SVC模式
	msr cpsr, r0			@ 将r0中的数据写入到cpsr寄存器中

	ldr sp, =0x80200000		@ 设置栈指针
	b main					@ 跳转到main函数运行
```