---
title: IMX6ULL学习笔记(四)
date: 2020-09-18 23:00:50
tags: [嵌入式, IMX6ULL, 学习笔记]
keywords:
description: 正点原子alpha开发板IMX6ULL裸机开发学习笔记。
---

**IMX6ULL裸机开发学习**

以下内容是我在学习正点原子`IMX6ULL`开发板`alpha`中记录的笔记，部分摘录自正点原子`IMX6ULL开发手册`。推荐看《跟我一起写Makefile》

## Makefile中变量的使用

变量在声明时需要给予初值，而在使用时，需要给在变量名前加上"`$`"符号，但最好
用小括号“`（）`”或是大括号“`{}`”把变量给包括起来。如果你要使用真实的“`$`”字符，
那么你需要用“`$$`”来表示。 

<!--more--> 

### 操作符`:=`

为了防止“`=`”在变量中使用变量会造成无限的变量展开，比如下面这种情况  

```makefile
A = ${B}
B = ${A}
```

所以常用`:=`操作符来定义变量。  

```makefile
x := foo
y := ${x} bar
x := later
# 等价于
y := foo bar
x := later
```

这种方法，前面的变量不能使用后面的变量，只能使用前面已定义好了的变
量。  比如

```makefile
y := ${x} bar
x := foo
```

那么`y`的值是`bar`，而不是`foo bar`

### 操作符`?=`

```makefile
FOO ?= bar
```

其含义是，如果`FOO`没有被定义过，那么变量`FOO`的值就是`bar`，否则，这条语句什么也不做，相当于  

```makefile
ifeq ($(origin FOO), undefined)
FOO = bar
endif
```

## Makefile练手

```makefile
# 使用的交叉编译器
CROSS_COMPILER 	?= 	arm-linux-gnueabihf-
CC 				:= 	$(CROSS_COMPILER)gcc
LD 				:= 	$(CROSS_COMPILER)ld
OBJCOPY 		:= 	$(CROSS_COMPILER)objcopy
OBJDUMP 		:= 	$(CROSS_COMPILER)objdump

# 链接生成的文件名
TARGET 			:= 	led

# Make查找路径
VPATH 			:=	project \
					imx6ul \
					bsp/led \
					bsp/clk \
					bsp/delay \

# 工程所有.C .S .H 文件所在路径
INCLUDEDIRS 	:= 	project \
					imx6ul \
					bsp/led \
					bsp/clk \
					bsp/delay \

# 给路径加上 -I 参数，因为编译时需要对路径需要使用 -I 选项
INCLUDES 		:=	$(patsubst %, -I %, $(INCLUDEDIRS))

# 查找工程下所有的 .s .c 文件，包含路径
SFILES 			:= 	$(foreach dir, $(INCLUDEDIRS), $(wildcard $(dir)/*.s))
CFILES 			:= 	$(foreach dir, $(INCLUDEDIRS), $(wildcard $(dir)/*.c))

# 所有的 .s .c 文件 去掉前面的路径
SFILESNODIR		:= 	$(notdir $(SFILES))
CFILESNODIR		:= 	$(notdir $(CFILES))

# 获取所有的 .s .c 文件的文件名，将其后缀名改为 .o ,并添加 obj/ 前缀,这样编译生成的.o文件就会放置到 obj文件夹下
SOBJS			:= 	$(addprefix obj/, $(SFILESNODIR:.s=.o))
COBJS			:= 	$(addprefix obj/, $(CFILESNODIR:.c=.o))

OBJS 			:=  $(SOBJS) $(COBJS)

$(TARGET).bin:$(OBJS)
	$(LD) -T imx6ul.lds -o $(TARGET).elf $^
	$(OBJCOPY) -O binary -S -g $(TARGET).elf $@
	$(OBJDUMP) -D -m arm $(TARGET).elf > $(TARGET).dis

# SOBJS中所有匹配 obj/%.o 的文件名 所对应的依赖 %.s
$(SOBJS) : obj/%.o : %.s
	$(CC) -c -O2 $(INCLUDES) -o $@ $< 

# COBJS中所有匹配 obj/%.o 的文件名 所对应的依赖 %.c
$(COBJS) : obj/%.o : %.c
	$(CC) -c -O2 $(INCLUDES) -o $@ $<

.PHONY : clean
clean:
	rm -rf obj/*.o $(TARGET).bin $(TARGET).elf $(TARGET).dis load.imx

.PHONY : printf
printf:
	@echo INCLUDES=$(INCLUDES)
	@echo SFILES=$(SFILES)
	@echo CFILES=$(CFILES)
	@echo SFILESNODIR=$(SFILESNODIR)
	@echo CFILESNODIR=$(CFILESNODIR)
	@echo SOBJS=$(SOBJS)
	@echo COBJS=$(COBJS)
	@echo OBJS=$(OBJS)
```

