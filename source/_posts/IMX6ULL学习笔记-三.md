---
title: IMX6ULL学习笔记(三)
date: 2020-09-18 22:58:51
tags: [嵌入式, IMX6ULL, 学习笔记]
keywords:
description: 正点原子alpha开发板IMX6ULL裸机开发学习笔记。
---

**IMX6ULL裸机开发学习**

以下内容是我在学习正点原子`IMX6ULL`开发板`alpha`中记录的笔记，部分摘录自正点原子`IMX6ULL开发手册`。

## 链接脚本

链接脚本的例子  

```
SECTIONS
{
	. = 0X87800000;
	.text :
	{
		start.o
		main.o
		*(.text)
	}
	.rodata ALIGN(4) : {*(.rodata)}
	.data ALIGN(4) : {*(.data)}
	__bss_start = .;
	.bss ALIGN(4) : {*(.bss) *(.COMMON)}
	__bss_end = .;
}
```

<!--more-->



## 加上清除BSS段，代码不运行

\_\_bss\_start = 0X87800289 。对于32位的SOC来说，一般访问是4字节访问的。0X0，0X4，0X8，0XC。芯片处理的时候以4字节访问，因此会从0X87800288开始清除BSS段。然而0X87800288不属于BSS段。所以我们需要对\_\_bss\_start进行四字节对齐。按照四字节对齐的原理，\_\_bss_start = 0X8780028C。所以需要设置\_\_bss\_start为四字节对齐。

```
SECTIONS 
{
    . = 0x87800000;
    .text : 
    {
        obj/start.o;
        *(.text);
    }
    .rodata ALIGN(4) : {*(.rodata)}
    .data ALIGN(4) : {*(.data)}
    . = ALIGN(4);
    __bss_start = .;
    .bss ALIGN(4) : {*(.bss) *(COMMON)}
    __bss_end = .;
}
```

