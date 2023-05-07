---
title: IMX6ULL学习笔记(一)
date: 2020-09-18 22:40:41
tags: [嵌入式, IMX6ULL, 学习笔记]
keywords:
description: 正点原子alpha开发板IMX6ULL裸机开发学习笔记。
---

**IMX6ULL裸机开发学习**

以下内容是我在学习正点原子`IMX6ULL`开发板`alpha`中记录的笔记，部分摘录自正点原子`IMX6ULL开发手册`。

# IMX6UL裸机开发bin文件头部信息分析

## IVT、Boot Data和DCD数据

Bin文件前面要添加头部（IVT+Boot Data+DCD数据），由官方手册可知要烧写到SD卡中的load.imx文件在SD卡中的起始地址是0x400，也就是1024.  

头部大小是3KB，加上偏移的1KB，一共是4KB，因此在SD卡中bin文件起始地址为4096。IVT大小为32B/4=8条。

<!--more-->

IVT数据格式：  

| IVT结构   | 数据       | 描述                                                         |
| --------- | ---------- | ------------------------------------------------------------ |
| header    | 0X402000D1 | IVT头部信息                                                  |
| entry     | 0X87800000 | 保存着程序入口地址，也就是镜像第一行指令所在的位置           |
| reserved1 | 0X00000000 | 保留，未使用                                                 |
| dcd       | 0x877FF42C | 保存着DCD数据的起始地址，0X87800000-0XC00(IVT+Boot Data+DCD=3KB)=0X877FF400(load.imx起始地址)，所以DCD相对于load.imx起始地址偏移了0X2C（44Byte,IVT=32Byte,Boot Bata=12Byte） |
| boot data | 0X877FF420 | 保存着Boot数据起始地址，IVT=32Byte，0X877FF400+0X20（32Byte）=0X877FF420 |
| self      | 0X877FF400 | IVT复制到DDR中以后的首地址                                   |
| csf       | 0X00000000 | CSF地址                                                      |
| reserved2 | 0X00000000 | 保留，未使用                                                 |

Boot Data数据格式：  

| Boot Data结构 | 数据       | 描述                                                 |
| ------------- | ---------- | ---------------------------------------------------- |
| start         | 0X877FF000 | 整个load.imx的起始地址，包括前面的1KByte地址偏移     |
| length        | 0X00200000 | 镜像大小，这里设置2MByte。因此镜像大小不能超过2MByte |
| plugin        | 0X00000000 | 插件                                                 |

DCD数据格式：   

| Header   （Tag+Length+Version） |
| ------------------------------- |
| [CMD]                           |
| [CMD]                           |
| ……                              |

DCD CMD数据格式：  

| Header    （Tag+Length+Parameter） |
| :--------------------------------: |
|              Address               |
|             Value/Mask             |
|             [Address]              |
|            [Value/Mask]            |
|                 ……                 |
|             [Address]              |
|            [Value/Mask]            |

DCD数据整体举例：  

| DCD结构                                                      | 数据       | 描述                                                         |
| ------------------------------------------------------------ | ---------- | ------------------------------------------------------------ |
| header                                                       | 0X40E801D2 | header 格式,第一个字节 Tag 为 0XD2,第二和三这两个字节为 DCD 大小,为大端模式,所以 DCD 大小为 0X01E8=488 字节。第四个字节为 0X40。 |
| Write Data Command                                           | 0X04E401CC | 第一个为 Tag,固定为 0XCC,第二和三这两个字节是大端模式的命令总长度,为 0X01E4=484 个字节。第四个字节是 Parameter,为 0X04,表示目标位置宽度为 4 个字节。 |
| Address                                                      | 0X020C4068 | 寄存器 CCGR0 地址                                            |
| Value                                                        | 0XFFFFFFFF | 要写入寄存器 CCGR0 的值,表示打开 CCGR0 控制的所有外设时钟。  |
| ……                                                           | ……         | CCGR1~CCGR5 这些寄存器的地址和值。                           |
| IVT、Boot Data和DCD数据Address                               | 0X020C4080 | 寄存器 CCGR0 地址                                            |
| Bin文件前面要添加头部（IVT+Boot Data+DCD数据），由官方手册可知要烧写到SD卡中的load.imx文件在SD卡中的起始地址是0x400，也就是1024.  Value | 0XFFFFFFFF | 要写入寄存器 CCGR6 的值,表示打开 CCGR6 控制的所有外设时钟。  |
| 头部大小是3KB，加上偏移的1KB，一共是4KB，因此在SD卡中bin文件起始地址为4096。IVT大小为32B/4=8条。…… | ……         | ……                                                           |
| IVT数据格式：  Check Data Command                            | ……         | ……                                                           |
| IVT结构                                                      | 数据……     | 描述……                                                       |

