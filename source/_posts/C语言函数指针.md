---
title: C语言函数指针
date: 2020-09-18 22:37:12
tags: [函数指针, 随笔]
keywords:
description: 遇到的一种函数指针的用法。
---

# C语言函数指针用法

```c
//定义中断处理函数类型
typedef void (*system_Irq_Handler_t)(unsigned int gicciar, void *param);

/** 
 * 定义一种新的变量类型，类型名 *system_Irq_Handler_t 因此用这个类型定义的变量是一个指针
 * 这种指针可以指向 void function(unsigned int gicciar, void *param) 这种类型的函数
 * 常用在函数数组中，这样可以通过函数数组来直接调用函数。
 * */
```