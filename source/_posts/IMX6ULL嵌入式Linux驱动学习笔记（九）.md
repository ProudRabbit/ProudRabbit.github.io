---
title: IMX6ULL嵌入式Linux驱动学习笔记（九）
date: 2021-01-08 11:19:03
tags: [嵌入式, IMX6ULL, 学习笔记]
keywords:
description: 正点原子alpha开发板IMX6ULL嵌入式Linux开发学习笔记。
---

**IMX6ULL嵌入式Linux驱动开发学习**

以下内容是我在学习正点原子`IMX6ULL`开发板`alpha`中记录的笔记，部分摘录自正点原子`IMX6ULL开发手册`。



## 一、Linux内核中断处理

### 1.1 裸机中断

- 使能中断，初始化相应的寄存器。
- 注册中断服务函数，也就是向`irqTable`数组（裸机例程）的指定标号处写入中断服务函数。
- 中断发生后进入`IRQ`中断服务函数，在`IRQ`中断服务函数中，根据中断号在`irqTable`里面查找具体的中断处理函数，找到以后执行相应的中断处理函数。

<!--more-->

### 1.2 Linux中断

- 先知道要使用的中断对应的中断号。

- 根据终端号申请`request_irq`，`request_irq`函数可能会导致睡眠，此函数同时会激活中断。

  > ```c
  > /* @param irq 中断号
  >  * @param handler 中断服务函数 
  >  * @param flags 中断标志、中断触发方式
  >  * @param name 中断名字
  >  * @param dev 使用共享中断时，唯一用来区分的标志。当多个设备共享一个中断线，共享的所有中断都必须指定此标志。
  >  * @return 0 中断申请成功，其他负值 中断申请失败，如果返回 -EBUSY 的话表示中断已经被申请了。
  >  */
  > int request_irq(unsigned int irq, irq_handler_t handler, 
  >                 unsigned long flags, const char *name, void *dev);
  > ```

- 不再使用中断时，需要释放中断`free_irq`。

  > ```c
  > /* @param irq 中断号
  >  * @param dev 如果中断设置为共享(IRQF_SHARED)的话，此参数用来区分具体的中断。
  >  * 			  共享中断只有在释放最后中断处理函数的时候才会被禁止掉。
  >  */
  > void free_irq(unsigned int irq, void *dev);
  > ```

- 在使用`request_irq`函数申请中断的时候需要设置中断处理函数，中断处理函数格式如下所示

  > ```c
  > irqreturn_t (*irq_handler_t) (int, void *)
  > ```
  >
  > 第一个参数是要中断处理函数要相应的中断号。第二个参数是一个指向 `void` 的指针，也就是个通用指针，需要与 `request_irq` 函数的 `dev` 参数保持一致。用于区分共享中断的不同设备，`dev` 也可以指向设备数据结构。中断处理函数的返回值为 `irqreturn_t` 类型， `irqreturn_t` 类型定义如下所示：
  >
  > ```c
  > enum irqreturn {
  >  IRQ_NONE = (0 << 0),
  >  IRQ_HANDLED = (1 << 0),
  >  IRQ_WAKE_THREAD = (1 << 1),
  > };
  > 
  > typedef enum irqreturn irqreturn_t;
  > ```
  >
  > `irqreturn_t`是个枚举类型，一共有三种返回值。一般中断服务函数返回值使用如下形式：
  >
  > `return IRQ_RETVAL(IRQ_HANDLED)` 

- 中断使能和禁止

  > `enable_irq` 和 `disable_irq` 用于使能和禁止指定的中断，`irq` 就是要禁止的中断号。
  >
  > ```c
  > void enable_irq(unsigned int irq);
  > void disable_irq(unsigned int irq);
  > ```
  >
  > `disable_irq`函数要等到当前正在执行的中断处理函数执行完才返回，因此需要保证不会产生新的中断，并且确保所有已经开始执行的中断处理程序已经全部退出。在这种情况下，可以使用另外一个中断禁止函数：
  >
  > ```c
  > void disable_irq_nosync(unsigned int irq)
  > ```
  >
  > `disable_irq_nosync` 函数调用以后立即返回，不会等待当前中断处理程序执行完毕。
  >
  > ---
  >
  > 上面三个函数都是使能或者禁止某一个中断，有时候我们需要关闭当前处理器的整个中断系统，也就是在学习 `STM32` 的时候常说的关闭全局中断，这个时候可以使用如下两个函数：
  >
  > ```c
  > local_irq_enable();
  > local_irq_disable();
  > ```
  >
  > `local_irq_enable` 用于使能当前处理器中断系统，`local_irq_disable`用于禁止当前处理器中断系统。
  >
  > ---
  >
  > 但是在任务中使用这两个函数会出现问题。比如假如 `A` 任务调用 `local_irq_disable` 关闭全局中断10秒，当关闭了2秒的时候 `B` 任务开始运行， `B` 任务也调用 `local_irq_disable` 关闭全局中断3秒， 3秒以后 `B` 任务调用 `local_irq_enable` 函数将全局中断打开了。此时才过去 2+3=5 秒的时间，然后全局中断就被打开了，此时 `A` 任务要关闭10秒全局中断的愿望就破灭了，然后 `A` 任务就“生气了”，结果很严重，可能系统都要被`A`任务整崩溃。为了解决这个问题， `B` 任务不能直接简单粗暴的通过`local_irq_enable` 函数来打开全局中断，而是将中断状态恢复到以前的状态，要考虑到别的任务的感受，此时就要用到下面两个函数：
  >
  > ```c
  > local_irq_save(flags);
  > local_irq_restore(flags);
  > ```
  >
  > 这两个函数是一对， `local_irq_save` 函数用于禁止中断，并且将中断状态保存在 `flags` 中。
  > `local_irq_restore` 用于恢复中断，将中断恢复到 `flags` 状态。

### 1.3 上半部和下半部

- 上半部：上半部就是中断处理函数，那些处理过程比较快，不会占用很长时间的处理就可以放在上半部完成。

- 下半部：如果中断处理过程比较耗时，那么就将这些比较耗时的代码提出来，交给下半部去执行，这样中断处理函数就会快进快出。  

- 关于代码属于上半部或下半部的参考

  > 1. 如果要处理的内容不希望被其他中断打断，那么可以放到上半部。  
  > 2. 如果要处理的任务对时间敏感，可以放到上半部。
  > 3. 如果要处理的任务与硬件有关，可以放到上半部。
  > 4. 除了上述三点以外的其他任务，优先考虑放到下半部。

---

**`Linux`对下半部的处理方式。**

#### 1.3.1 软中断

**软中断必须在编译的时候静态注册（写入到内核中）！软中断不要去用。**

`Linux` 内核使用结构体`softirq_action` 表示软中断，`softirq_action`结构体定义在文件 `include/linux/interrupt.h` 中，内容如下：

```c
struct softirq_action
{
	void (*action)(struct softirq_action *);
};
```

在 `kernel/softirq.c` 文件中一共定义了 10 个软中断，如下所示：

```c
static struct softirq_action softirq_vec[NR_SOFTIRQS];
```

`NR_SOFTIRQS` 是枚举类型，定义在文件 `include/linux/interrupt.h` 中，定义如下：

```c
enum 
{
    HI_SOFTIRQ = 0, 			/* 高优先级软中断 */
    TIMER_SOFTIRQ, 				/* 定时器软中断 */
    NET_TX_SOFTIRQ, 			/* 网络数据发送软中断 */
    NET_RX_SOFTIRQ,				/* 网络数据接收软中断 */
    BLOCK_SOFTIRQ,
    BLOCK_IOPOLL_SOFTIRQ,
    TASKLET_SOFTIRQ,			/* tasklet 软中断 */
    SCHED_SOFTIRQ,				/* 调度软中断 */
    HRTIMER_SOFTIRQ,			/* 高精度定时器软中断 */
    RCU_SOFTIRQ,				/* RCU 软中断 */
    NR_SOFTIRQS
};
```

`softirq_action` 结构体中的 `action` 成员变量就是软中断的服务函数，数组 `softirq_vec` 是个全局数组，因此所有的 `CPU`(对于 `SMP` 系统而言)都可以访问到，每个 `CPU` 都有自己的触发和控制机制，并且只执行自己所触发的软中断。但是各个 `CPU` 所执行的软中断服务函数确是相同的，都是数组 `softirq_vec` 中定义的 `action` 函数。

---

- 要使用软中断要先注册。

  > ```c
  > /* @brief 注册软中断服务函数
  >  * @param nr 要开启的软中断 是上面枚举中的一个。
  >  * @param action 软中断对应的处理函数
  >  */
  > void open_softirq(int nr, void (*action)(struct softirq_action *))
  > ```
  >
  > 软中断必须在编译的时候静态注册！

- 触发软中断

  > 注册好软中断以后需要通过 `raise_softirq` 函数触发， `raise_softirq` 函数原型如下：
  >
  > ```c
  > /* @brief 触发软中断
  >  * @param nr 要触发的软中断
  >  */
  > void raise_softirq(unsigned int nr)
  > ```

#### 1.3.2 tasklet

`tasklet` 是利用软中断来实现的另外一种下半部机制，建议使用 `tasklet`。

```c
struct tasklet_struct
{
    struct tasklet_struct *next; 	/* 下一个 tasklet */
    unsigned long state; 			/* tasklet 状态 */
    atomic_t count; 				/* 计数器，记录对 tasklet 的引用数 */
    void (*func)(unsigned long);	/* tasklet 执行的函数 */
    unsigned long data;				/* 函数 func 的参数 */
};
```

如果要使用 `tasklet`，必须先定义一个 `tasklet`，然后使用 `tasklet_init` 函数初始化 `tasklet`，`taskled_init` 函数原型如下：

```c
/* @brief 初始化 tasklet
 * @param t 要初始化的 tasklet
 * @param func tasklet 的处理函数。
 * @param data 要传递给 func 函数的参数
 */
void tasklet_init(struct tasklet_struct *t, void (*func)(unsigned long), unsigned long data);
```

也 可 以 使 用 宏 `DECLARE_TASKLET` 来 一 次 性 完 成 `tasklet` 的 定 义 和 初 始 化 ，`DECLARE_TASKLET` 定义在 `include/linux/interrupt.h` 文件中，定义如下：

```c
/* name 为要定义的 tasklet 名字，
 * 这个名字就是一个 tasklet_struct 类型的结构变量，
 * func 就是 tasklet 的处理函数，data 是传递给 func 函数的参数。
 */
DECLARE_TASKLET(name, func, data);
```

在上半部，也就是中断处理函数中调用 `tasklet_schedule` 函数就能使 `tasklet` 在合适的时间运
行， `tasklet_schedule` 函数原型如下：

```c
/* @param t 要调度的 tasklet，也就是 DECLARE_TASKLET 宏里面的 name。*/
void tasklet_schedule(struct tasklet_struct *t)；
```

---

**`tasklet` 使用顺序：**

1. 定义一个`tasklet`。
2. 初始化`tasklet`，重点是设置对应的处理函数。
3. 在上半部中调用`tasklet_schedule`函数，使 `tasklet` 在合适的时间运行。

```c
/* 定义 taselet */
struct tasklet_struct testtasklet;

/* tasklet 处理函数 */
void testtasklet_func(unsigned long data)
{
	/* tasklet 具体处理内容 */
}

/* 中断处理函数 */
irqreturn_t test_handler(int irq, void *dev_id)
{
    ......
    /* 调度 tasklet */
    tasklet_schedule(&testtasklet);
    ......
}

/* 驱动入口函数 */
static int __init xxxx_init(void)
{
    ......
    /* 初始化 tasklet */
    tasklet_init(&testtasklet, testtasklet_func, data);
    /* 注册中断处理函数 */
    request_irq(xxx_irq, test_handler, 0, "xxx", &xxx_dev);
    ......
}
```

#### 1.3.3 工作队列

工作队列是另外一种下半部执行方式，工作队列在进程上下文执行，工作队列将要推后的工作交给一个内核线程去执行，因为工作队列工作在进程上下文，因此工作队列允许睡眠或重新调度。因此如果要推后的工作可以睡眠那么就可以选择工作队列，否则的话就只能选择软中断或 `tasklet`。

`Linux` 内核使用 `work_struct` 结构体表示一个工作，内容如下(省略掉条件编译)：

```c
struct work_struct {
    atomic_long_t data;
    struct list_head entry;
    work_func_t func; /* 工作队列处理函数 */
};
```

这些工作组织成工作队列，工作队列使用 `workqueue_struct` 结构体表示，内容如下(省略掉条件编译)：

```c
struct workqueue_struct {
    struct list_head pwqs;
    struct list_head list;
    struct mutex mutex;
    int work_color;
    int flush_color;
    atomic_t nr_pwqs_to_flush;
    struct wq_flusher *first_flusher;
    struct list_head flusher_queue;
    struct list_head flusher_overflow;
    struct list_head maydays;
    struct worker *rescuer;
    int nr_drainers;
    int saved_max_active;
    struct workqueue_attrs *unbound_attrs;
    struct pool_workqueue *dfl_pwq;
    char name[WQ_NAME_LEN];
    struct rcu_head rcu;
    unsigned int flags ____cacheline_aligned;
    struct pool_workqueue __percpu *cpu_pwqs;
    struct pool_workqueue __rcu *numa_pwq_tbl[];
};
```

`Linux` 内核使用工作者线程`(worker thread)`来处理工作队列中的各个工作， `Linux` 内核使用`worker` 结构体表示工作者线程， `worker` 结构体内容如下：

```c
struct worker {
    union {
        struct list_head entry;
        struct hlist_node hentry;
    };
    struct work_struct *current_work;
    work_func_t current_func;
    struct pool_workqueue *current_pwq;
    bool desc_valid;
    struct list_head scheduled;
    struct task_struct *task;
    struct worker_pool *pool;
    struct list_head node;
    unsigned long last_active;
    unsigned int flags;
    int id;
    char desc[WORKER_DESC_LEN];
    struct workqueue_struct *rescue_wq;
};
```

每个 `worker` 都有一个工作队列，工作者线程处理自己工作队列中的所有工作。在实际的驱动开发中，只需要定义工作`(work_struct)`即可，关于工作队列和工作者线程基本不用去管。

- 创建工作直接定一个`work_struct`结构体，然后使用`INIT_WORK`宏来初始化工作即可，`INIT_WORK`宏定义如下：

  >   ```c
  >   /* _work 表示要初始化的工作， _func 是工作对应的处理函数。*/
  >   #define INIT_WORK(_work, _func)
  >   ```
  >
  >   也可以使用`DECLARE_WORK`宏一次性完成工作的创建和初始化，宏定义如下：
  >
  >   ```c
  >   /* n 表示定义的工作(work_struct)， f 表示工作对应的处理函数。*/
  >   #define DECLARE_WORK(n, f)
  >   ```

- 同`tasklet`一样，工作也是需要调度才能运行的，工作的调度函数为 `schedule_work`，函数原型如下所示：

  >   ```c
  >   /* @brief 调度工作
  >   * @param work 要调度的工作
  >   * @return 结果 0 成功，其他值 失败
  >   */
  >   bool schedule_work(struct work_struct *work)
  >   ```


---

**工作队列使用顺序：**

1. 定义一个`work`。
2. 初始化`work`，重点同样是是设置对应的处理函数。
3. 在上半部中调用`schedule_work`函数，使 `work` 在合适的时间运行。

```c
/* 定义工作(work) */
struct work_struct testwork;

/* work 处理函数 */
void testwork_func_t(struct work_struct *work);
{
    /* 根据成员变量反推出结构体的首地址 */
    //struct demo_struct *p = container_of(work, struct demo_struct, 成员变量名);
	/* work 具体处理内容 */
}

/* 中断处理函数 */
irqreturn_t test_handler(int irq, void *dev_id)
{
    ......
    /* 调度 work */
    schedule_work(&testwork);
    ......
}

/* 驱动入口函数 */
static int __init xxxx_init(void)
{
    ......
    /* 初始化 work */
    INIT_WORK(&testwork, testwork_func_t);
    /* 注册中断处理函数 */
    request_irq(xxx_irq, test_handler, 0, "xxx", &xxx_dev);
    ......
}
```

### 1.4 设备树中断节点信息

`#interrupt-cells`指定中断域编码中断说明符所需的单元数。对于设备来说，会使用`interrupts`属性来描述中断信息，而`#interrupt-cells`则指定了描述一个中断信息需要几个数值。

> ```c
> intc: interrupt-controller@00a01000 {
>  compatible = "arm,cortex-a7-gic";
>  #interrupt-cells = <3>;
>  interrupt-controller;
>  reg = <0x00a01000 0x1000>,
>  		<0x00a02000 0x100>;
> };
> 
> gpio5: gpio@020ac000 {
>  compatible = "fsl,imx6ul-gpio", "fsl,imx35-gpio";
>  reg = <0x020ac000 0x4000>;
>  interrupts = <GIC_SPI 74 IRQ_TYPE_LEVEL_HIGH>,
>  			 <GIC_SPI 75 IRQ_TYPE_LEVEL_HIGH>;
>  gpio-controller;
>  #gpio-cells = <2>;
>  interrupt-controller;		// 表示当前节点是中断控制器
>  #interrupt-cells = <2>;
> };
> ```
>
> 从`gpio5`的`interrupts`属性可以看到描述一个中断信息需要三个`cell`，分别是
>
> - 第一个 `cell`：中断类型， `0` 表示 `SPI` 中断（共享中断，不是 `SPI`通讯的中断）， `1` 表示 `PPI` 中断。
> - 第二个 `cell`：中断号，对于 `SPI` 中断来说中断号的范围为 `0~987`，对于 `PPI` 中断来说中断号的范围为 `0~15`。
> - 第三个 `cell`：标志， `bit[3:0]`表示中断触发类型，为 `1` 的时候表示上升沿触发，为 `2` 的时候表示下降沿触发，为 `4` 的时候表示高电平触发，为 `8` 的时候表示低电平触发。`bit[15:8]`为 `PPI` 中断的 `CPU 掩码`。

---

在NXP的官方`6ull`开发板上有一个磁力计芯片`fxls8471`，`fxls8471`的中断引脚链接到`I.MX6ULL`的`SNVS_TAMPER0`引脚上，而这个引脚可以复用为`GPIO_IO0`。在设备树中`fxls8471`的描述如下：

```c
fxls8471@1e {
    compatible = "fsl,fxls8471";
    reg = <0x1e>;
    position = <0>;
    interrupt-parent = <&gpio5>;
    interrupts = <0 8>;
};  
```

- `interrupt-parent` 属性设置中断控制器，这里使用 `gpio5` 作为中断控制器。
- `interrupts` 属性设置中断信息，因为在上面 `gpio5` 的节点中将 `#interrupt-cells` 设置为了`2`，所以这里的 `interrupts` 属性，需要使用两个 `cell` 来描述中断信息。

---

从设备树中获取中断号

```c
/* @brief 获取中断号
 * @param dev 设备节点
 * @param index 索引号
 * @return 中断号
 */
unsigned int irq_of_parse_and_map(struct device_node *dev, int index)
```

如果使用 `GPIO` 的话，可以使用 `gpio_to_irq` 函数来获取 `gpio` 对应的中断号，函数原型如下：

```c
/* @brief 获取gpio对应的中断号
 * @param gpio 要获取的GPIO编号
 * @return GPIO对应的中断号
 */
int gpio_to_irq(unsigned int gpio)
```

## 二、编写按键中断实验驱动

### 2.1 配置设备树

```c
/dts-v1/;

#include <dt-bindings/input/input.h>
#include "imx6ull.dtsi"

/ {
	model = "Freescale i.MX6 ULL 14x14 EVK Board";
	compatible = "fsl,imx6ull-14x14-evk", "fsl,imx6ull";

	……

	/* 自己添加的节点 2020-9-17 */

	……

	key {
		compatible = "atkalpha,key";
		pinctrl-names = "default";
		pinctrl-0 = <&pinctrl_key>;
		key-gpios = <&gpio1 18 GPIO_ACTIVE_HIGH>;
		status = "okay";
		interrupt-parent = <&gpio1>;
		interrupts = <18 IRQ_TYPE_EDGE_BOTH>;
	};
};

……

&iomuxc {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_hog_1>;
	imx6ul-evk {
		pinctrl_hog_1: hoggrp-1 {
			fsl,pins = <
				MX6UL_PAD_UART1_RTS_B__GPIO1_IO19		0x17059 /* SD1 CD */
				MX6UL_PAD_GPIO1_IO05__USDHC1_VSELECT	0x17059 /* SD1 VSELECT */
				MX6UL_PAD_GPIO1_IO09__GPIO1_IO09        0x17059 /* SD1 RESET */
			>;
		};

		/* 自己定义的IO复用 START */

		……

		pinctrl_key: keygrp {
			fsl,pins = <
				MX6UL_PAD_UART1_CTS_B__GPIO1_IO18	0XF080
			>;
		};

		/* 自己定义的IO复用 END */

		……
	};
};

……
```

### 2.2 按键中断驱动程序

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
	
};

struct keyirq_dev keyirq;		/* 定义一个keyirq设备 */

static int keyirq_open(struct inode *inode, struct file *filp)
{
	filp->private_data = &keyirq;
	
	return 0;
}	

static int keyirq_release(struct inode *inode, struct file *filp)
{
	struct keyirq_dev *dev = filp->private_data;
	
	return 0;
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
	.owner = THIS_MODULE,
	.open = keyirq_open,
	.release = keyirq_release,
	.write = keyirq_write,
	.read = keyirq_read,
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
}

/**
 * @brief 按键中断处理函数
 * @param irq 中断号
 */
static irqreturn_t key0_handler(int irq, void *arg)
{
	struct keyirq_dev *dev = arg;

	/* 启动定时器来延时消抖 */
	dev->timer.data = (volatile unsigned long)arg;
	mod_timer(&dev->timer, jiffies + msecs_to_jiffies(20));
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

		/* 获取中断号 */
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

### 2.3 使用下半部`tasklet`的按键中断驱动程序

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
	
};

struct keyirq_dev keyirq;		/* 定义一个keyirq设备 */

static int keyirq_open(struct inode *inode, struct file *filp)
{
	filp->private_data = &keyirq;
	
	return 0;
}	

static int keyirq_release(struct inode *inode, struct file *filp)
{
	struct keyirq_dev *dev = filp->private_data;
	
	return 0;
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
	.owner = THIS_MODULE,
	.open = keyirq_open,
	.release = keyirq_release,
	.write = keyirq_write,
	.read = keyirq_read,
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

### 2.4 测试APP程序

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/ioctl.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

/**
 * ./keyReadAPP /dev/keyirq
 * @param argc 应用程序参数个数
 * @param argv 保存的参数，字符串形式。
 * */
int main(int argc, char *argv[])
{
	int ret = 0;
	int fd = 0;
	char *filename;
	unsigned char data;

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

	/* 循环读取 */
	while (1)
	{
		ret = read(fd, &data, sizeof(data));
		if (ret < 0)
		{ 
			
		}
		else
		{
			printf("keyValue = %#x\r\n", data);
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

