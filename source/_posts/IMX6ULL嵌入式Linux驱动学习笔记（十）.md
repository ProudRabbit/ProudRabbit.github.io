---
title: IMX6ULL嵌入式Linux驱动学习笔记（十）
date: 2021-01-08 11:21:09
tags: [嵌入式, IMX6ULL, 学习笔记]
keywords:
description: 正点原子alpha开发板IMX6ULL嵌入式Linux开发学习笔记。
---

**IMX6ULL嵌入式Linux驱动开发学习**

以下内容是我在学习正点原子`IMX6ULL`开发板`alpha`中记录的笔记，部分摘录自正点原子`IMX6ULL开发手册`。



## 一、Linux阻塞和非阻塞IO

### 1.1 阻塞和非阻塞简介

这里的 `IO` 指的是 `Input/Output`，也就是输入/输出，是应用程序对驱动设备的输入/输出操作。当应用程序对设备驱动进行操作的时候，如果不能获取到设备资源，那么阻塞式 `IO` 就会将应用程序对应的线程挂起，直到设备资源可以获取为止。对于非阻塞 `IO`，应用程序对应的线程不会挂起，它要么一直轮询等待，直到设备资源可以使用，要么就直接放弃。  

<!--more-->

- 阻塞式`IO`：

  > 当资源不可用时，应用程序就会挂起。当资源可用的时候，唤醒任务。应用程序使用`open`打开驱动文件，默认是阻塞方式打开。
  >
  > 阻塞式`IO`访问示意图：
  >
  > ![阻塞式IO](https://i.loli.net/2020/12/23/Hb3IhCd95umSLcD.png "阻塞式IO")

- 非阻塞式`IO`：

  > 当资源不可用的时候，应用程序轮询查看，或放弃。会有超时处理。应用程序如果想在使用`open`打开驱动文件时使用非阻塞的方式打开，需要使用`O_NONBLOCK`。
  >
  > 非阻塞式`IO`访问示意图：
  >
  > ![非阻塞式IO](https://i.loli.net/2020/12/23/TPzJBZXh7eqCg6V.png "非阻塞式IO")

### 1.2 等待队列（阻塞访问）

1. 等待队列头

   阻塞访问最大的好处就是当设备文件不可操作的时候进程可以进入休眠态，这样可以将`CPU` 资源让出来。但是，当设备文件可以操作的时候就必须唤醒进程，一般在中断函数里面完成唤醒工作。 `Linux`内核提供了等待队列(`wait queue`)来实现阻塞进程的唤醒工作，如果我们要在驱动中使用等待队列，必须创建并初始化一个等待队列头，等待队列头使用结构体`wait_queue_head_t` 表示：

   ```c
   struct __wait_queue_head {
       spinlock_t lock;
       struct list_head task_list;
   };
   typedef struct __wait_queue_head wait_queue_head_t;
   ```

   定义好等待队列头以后需要初始化，使用 `init_waitqueue_head` 函数初始化等待队列头，函数原型如下：

   ```c
   /**
    * @param q 等待队列头
    */
   void init_waitqueue_head(wait_queue_head_t *q)
   ```

   也可以使用宏 `DECLARE_WAIT_QUEUE_HEAD` 来一次性完成等待队列头的定义和初始化。  

2. 等待队列项

   等待队列头就是一个等待队列的头部，每个访问设备的进程都是一个队列项，当设备不可用的时候就要将这些进程对应的等待队列项添加到等待队列里面。结构体 `wait_queue_t` 表示等待队列项。

   ```c
   struct __wait_queue {
       unsigned int flags;
       void *private;
       wait_queue_func_t func;
       struct list_head task_list;
   };
   typedef struct __wait_queue wait_queue_t;
   ```

   同样可以使用宏 `DECLARE_WAITQUEUE` 定义并初始化一个等待队列项。

   ```c
   /**
    * @param name 等待队列项的名字
    * @param tsk 表示该等待队列项属于哪个任务（进程），一般为 current
    */
   DECLARE_WAITQUEUE(name, tsk)
   ```

3. 添加/移除等待队列项

   当设备不可访问的时候就需要将进程对应的等待队列项添加到前面创建的等待队列头中，只有添加到等待队列头中以后进程才能进入休眠态。当设备可以访问以后再将进程对应的等待队列项从等待队列头中移除即可。

   ```c
   /**
    * @brief 添加等待队列项到等待队列头
    * @param q 等待队列头
    * @param wait 等待队列项
    */
   void add_wait_queue(wait_queue_head_t *q,wait_queue_t *wait);
   
   /**
    * @brief 将等待队列项从等待队列头中移除
    * @param q 等待队列头
    * @param wait 等待队列项
    */
   void remove_wait_queue(wait_queue_head_t *q,wait_queue_t *wait)
   ```

4. 等待唤醒

   当设备可以使用的时候就要唤醒进入休眠态的进程，唤醒可以使用如下两个函数。

   ```c
   /**
    * @brief 唤醒等待队列头下所有的进程
    * @param q 要唤醒的等待队列头
    */
   void wake_up(wait_queue_head_t *q);
   void wake_up_interruptible(wait_queue_head_t *q);
   ```

   **`wake_up` 函数可以唤醒处于 `TASK_INTERRUPTIBLE` 和 `TASK_UNINTERRUPTIBLE` 状态的进程，而 `wake_up_interruptible` 函数只能唤醒处于 `TASK_INTERRUPTIBLE` 状态的进程。**

5. 等待事件

   除了主动唤醒以外，也可以设置等待队列等待某个事件，当这个事件满足以后就自动唤醒等待队列中的进程，和等待事件有关的`API`函数如下。

   | 函数                                                    | 描述                                                         |
   | ------------------------------------------------------- | ------------------------------------------------------------ |
   | wait_event(wq, condition)                               | 等待以 `wq` 为等待队列头的等待队列被唤醒，前提是 `condition` 条件必须满足(为真)，否则一直阻塞。此 函 数 会 将 进 程 设 置 为`TASK_UNINTERRUPTIBLE` 状态。 |
   | wait_event_timeout(wq, condition, timeout)              | 功能和 `wait_event` 类似，但是此函数可以添加超时时间，以 `jiffies` 为单位。此函数有返回值，如果返回 `0` 的话表示超时时间到，而且 `condition` 为假。为 `1` 的话表示 `condition` 为真，也就是条件满足了。 |
   | wait_event_interruptible(wq, condition)                 | 与 `wait_event` 函数类似，但是此函数将进程设置为`TASK_INTERRUPTIBLE`，就是可以被信号打断。 |
   | wait_event_interruptible_timeout(wq,condition, timeout) | 与 `wait_event_timeout` 函数类似，此函数也将进 程设置为`TASK_INTERRUPTIBLE`，可以被信号打断。 |

### 1.3 轮询（非阻塞访问）

如果用户应用程序以非阻塞的方式访问设备，设备驱动程序就要提供非阻塞的处理方式，也就是轮询。 `poll`、 `epoll` 和 `select` 可以用于处理轮询，应用程序通过 `select`、 `epoll` 或 `poll` 函数来查询设备是否可以操作，如果可以操作的话就从设备读取或者向设备写入数据。

当应用程序调用 `select`、 `epoll` 或 `poll` 函数的时候，设备驱动程序中的 `poll` 函数就会执行，因此需要在设备驱动程序中编写 `poll` 函数。

驱动里`poll`函数原型如下。

```c
/**
 * @param filp 要打开的设备文件（文件描述符）
 * @param wait 结构体 poll_table_struct 类型指针，由应用程序传递进来。一般将此参数传递给 poll_wait 函数。
 * @return 向应用程序返回设备或者资源状态。
 */
unsigned int (*poll)(struct file *filp, struct poll_table_struct *wait);
```

```c
/* return 可以返回的资源状态 */
POLLIN 			// 有数据可以读取。
POLLPRI 		// 有紧急的数据需要读取。
POLLOUT 		// 可以写数据。
POLLERR			// 指定的文件描述符发生错误。
POLLHUP			// 指定的文件描述符挂起。
POLLNVAL		// 无效的请求。
POLLRDNORM		// 等同于 POLLIN，普通数据可读
```

需要在驱动程序的 `poll` 函数中调用 `poll_wait` 函数， `poll_wait` 函数不会引起阻塞，只是将应用程序添加到 `poll_table` 中。

```c
/**
 * @param filp 文件描述符
 * @param wait_address 要添加到 poll_table 中的等待队列头
 * @param p poll_table，就是file_operations 中 poll 函数的 wait 参数
 */
void poll_wait(struct file *filp, wait_queue_head_t *wait_address, poll_table *p)
```

1. `select`函数

   >```c
   >/**
   >* @param nfds 所要监视的这三类文件描述集合中，最大文件描述符加1。
   >* @param readfds 监视指定描述符集的读变化，也就是监视文件是否可读
   >* @param writefds 监视文件是否可以进行写操作
   >* @param exceptfds 监视这些文件的异常
   >* @param timeout 超时时间，为 NULL 的时候就表示无限期的等待。
   >* @return 0，表示的话就表示超时发生，但是没有任何文件描述符可以进行操作；-1，发生错误；其他值，可以进行操作的文件描述符个数。
   >*/
   >int select(int nfds, fd_set *readfds, fd_set *writefds, 
   >          fd_set *exceptfds, struct timeval *timeout);
   >
   >/* timeval结构体 */
   >struct timeval {
   >   long tv_sec; 	/* 秒 */
   >   long tv_usec; 	/* 微妙 */
   >};
   >```
   >
   >比如现在要从一个设备文件中读取数据，那么就可以定义一个 `fd_set` 变量，这个变量要传递给参数 `readfds`。当我们定义好一个 `fd_set` 变量以后可以使用如下所示几个宏进行操作
   >
   >```c
   >void FD_ZERO(fd_set *set);
   >void FD_SET(int fd, fd_set *set);
   >void FD_CLR(int fd, fd_set *set);
   >int FD_ISSET(int fd, fd_set *set);
   >```
   >
   >`FD_ZERO` 用于将 `fd_set` 变量的所有位都清零， `FD_SET` 用于将 `fd_set` 变量的某个位置 `1`，也就是向 `fd_set` 添加一个文件描述符，参数 `fd` 就是要加入的文件描述符。`FD_CLR` 用于将`fd_set `变量的某个位清零，也就是将一个文件描述符从 `fd_set`中删除，参数 `fd` 就是要删除的文件描述符。 `FD_ISSET` 用于测试一个文件是否属于某个集合，参数 `fd` 就是要判断的文件描述符。
   >
   >使用 `select` 函数对某个设备驱动文件进行读非阻塞访问的操作示例如下
   >
   >```c
   >void main(void)
   >{
   >   int ret, fd; 				/* 要监视的文件描述符 */
   >   fd_set readfds; 				/* 读操作文件描述符集 */
   >   struct timeval timeout;		/* 超时结构体 */
   >
   >   fd = open("dev_xxx", O_RDWR | O_NONBLOCK); /* 非阻塞式访问 */
   >
   >   FD_ZERO(&readfds); 		/* 清除 readfds */
   >   FD_SET(fd, &readfds); 	/* 将 fd 添加到 readfds 里面 */
   >
   >   /* 构造超时时间 */
   >   timeout.tv_sec = 0;
   >   timeout.tv_usec = 500000; /* 500ms */
   >
   >   ret = select(fd + 1, &readfds, NULL, NULL, &timeout);
   >   switch (ret) {
   >       case 0: 		/* 超时 */
   >           printf("timeout!\r\n");
   >           break;
   >       case -1: 	/* 错误 */
   >           printf("error!\r\n");
   >           break;
   >       default: 	/* 可以读取数据 */
   >           if(FD_ISSET(fd, &readfds)) { /* 判断是否为 fd 文件描述符 */
   >           	/* 使用 read 函数读取数据 */
   >           }
   >           break;
   >   }
   >}
   >```

2. `poll`函数

   > 在单个线程中， `select` 函数能够监视的文件描述符数量有最大的限制，一般为 `1024`，可以修改内核将监视的文件描述符数量改大，但是这样会降低效率！这个时候就可以使用 `poll` 函数， `poll` 函数本质上和 `select` 没有太大的差别，但是 `poll` 函数没有最大文件描述符限制。
   >
   > ```c
   > /**
   >  * @param fds 要监视的文件描述符集合以及要监视的事件,为一个数组
   >  * @param nfds poll函数要监视的文件描述符数量
   >  * @param timeout 超时时间，单位为 ms。
   >  * @return 返回 revents 域中不为 0 的 pollfd 结构体个数，也就是发生事件或错误的文件描述符数量；0，超时；-1，发生错误，并且设置 errno 为错误类型
   >  */
   > int poll(struct pollfd *fds, nfds_t nfds, int timeout);
   > 
   > /* pollfd 结构体 */
   > struct pollfd {
   >     int fd; 		/* 文件描述符 */
   >     short events; 	/* 请求的事件 */
   >     short revents; 	/* 返回的事件 */
   > };
   > ```
   >
   > `pollfd` 结构体中 `fd` 是要监视的文件描述符，如果 `fd` 无效的话那么 `events` 监视事件也就无效，并且 `revents`返回 `0`。 `events`是要监视的事件，可监视的事件类型如下所示， `revents` 是返回参数，也就是返回的事件， 由 `Linux` 内核设置具体的返回事件  
   >
   > ```c
   > POLLIN 			/* 有数据可以读取。*/
   > POLLPRI 		/* 有紧急的数据需要读取。 */
   > POLLOUT 		/* 可以写数据。 */
   > POLLERR 		/* 指定的文件描述符发生错误。 */
   > POLLHUP 		/* 指定的文件描述符挂起。 */
   > POLLNVAL 		/* 无效的请求。 */
   > POLLRDNORM 		/* 等同于 POLLIN */
   > ```
   >
   > 使用 `poll` 函数对某个设备驱动文件进行读非阻塞访问的操作示例如下
   >
   > ```c
   > void main(void)
   > {
   >     int ret;
   >     int fd;		/* 要监视的文件描述符 */
   >     struct pollfd fds;
   > 
   >     fd = open(filename, O_RDWR | O_NONBLOCK);	/* 非阻塞式访问 */
   > 
   >     /* 构造结构体 */
   >     fds.fd = fd;
   >     fds.events = POLLIN;		/* 监视数据是否可以读取 */
   > 
   >     ret = poll(&fds, 1, 500);	/* 轮询文件是否可操作，超时 500ms */
   >     if (ret) { 				/* 数据有效 */
   >         ......
   >         /* 读取数据 */
   >         ......
   >     } else if (ret == 0) {	/* 超时 */
   >     	......
   > 
   >     } else if (ret < 0) {	/* 错误 */
   >     	......
   > 
   >     }
   > }
   > ```

3. `epoll`函数

   >传统的 `selcet` 和 `poll` 函数都会随着所监听的 `fd` 数量的增加，出现效率低下的问题，而且 `poll` 函数每次必须遍历所有的描述符来检查就绪的描述符，这个过程很浪费时间。为此， `epoll` 应运而生， `epoll` 就是为处理大并发而准备的，一般常常在网络编程中使用 `epoll` 函数。应用程序需要先使用 `epoll_create` 函数创建一个 `epoll` 句柄。
   >
   >```c
   >/**
   > * @param size 从Linux2.6.8开始此参数已无意义，随便填写一个大于0的值即可。
   > * @return epoll句柄，如果为-1的话表示创建失败。
   > */
   >int epoll_create(int size)；
   >```
   >
   >`epoll` 句柄创建成功以后使用 `epoll_ctl` 函数向其中添加要监视的文件描述符以及监视的事件。
   >
   >```c
   >/**
   > * @param epfd 要操作的epoll句柄，也就是使用epoll_create函数创建的epoll句柄。
   > * @param op 表示要对 epfd(epoll 句柄)进行的操作。
   > * @param fd 要监视的文件描述符。
   > * @param event 要监视的事件类型。
   > * @return 0，成功； -1，失败，并且设置 errno 的值为相应的错误码。
   > */
   >int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
   >
   >/* op 可以选择的设置  */
   >EPOLL_CTL_ADD // 向 epfd 添加文件参数 fd 表示的描述符。
   >EPOLL_CTL_MOD // 修改参数 fd 的 event 事件。
   >EPOLL_CTL_DEL // 从 epfd 中删除 fd 描述符。
   >
   >/* epoll_event 结构体 */
   >struct epoll_event {
   >   uint32_t events;	/* epoll 事件 */
   >   epoll_data_t data;	/* 用户数据 */
   >};
   >```
   >
   >结构体 `epoll_event` 的 `events` 成员变量表示要监视的事件，可选的事件如下，彼此之间可以进行或操作：
   >
   >```c
   >EPOLLIN			// 有数据可以读取。
   >EPOLLOUT		// 可以写数据。
   >EPOLLPRI		// 有紧急的数据需要读取。
   >EPOLLERR		// 指定的文件描述符发生错误。
   >EPOLLHUP		// 指定的文件描述符挂起。
   >EPOLLET			// 设置 epoll 为边沿触发，默认触发模式为水平触发。
   >EPOLLONESHOT	// 一次性的监视，当监视完成以后还需要再次监视某个 fd，那么就需要将 fd 重新添加到 epoll 里面。
   >```
   >
   >一切都设置好以后应用程序就可以通过 `epoll_wait` 函数来等待事件的发生，类似 `select` 函数。
   >
   >```c
   >/**
   > * @param epfd 要等待的 epoll
   > * @param events 指向 epoll_event 结构体的数组，当有事件发生的时候 Linux 内核会填写 events，调
   >用者可以根据 events 判断发生了哪些事件。
   > * @param maxevents events 数组大小，必须大于0
   > * @param timeout 超时时间，单位为ms。
   > * @return 0，超时；-1，错误；其他值，准备就绪的文件描述符数量。
   > */
   >int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
   >```
   >
   >**`epoll` 更多的是用在大规模的并发服务器上，因为在这种场合下 `select` 和 `poll` 并不适合，当设计到的文件描述符(`fd`)比较少的时候就适合用 `selcet` 和 `poll`。**

## 二、编写试验驱动

### 2.1 阻塞式访问驱动

1. 等待事件

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
	
	wait_queue_head_t r_wait;			/* 读等待队列头 */
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

	/* 等待事件 */
	wait_event_interruptible(dev->r_wait, atomic_read(&dev->keyRelease));
	
	// NOTE:wait_event 不可以被信号打断，使用 kill -9 PID 无法杀掉任务
	// wait_event(dev->r_wait, atomic_read(&dev->keyRelease));		/* 等待按键值有效 */


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

	/* 唤醒进程 */
	if(atomic_read(&dev->keyRelease))
	{
		wake_up(&dev->r_wait);
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

	/* 初始化等待队列头 */
	init_waitqueue_head(&keyirq.r_wait);

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

2. 等待队列项

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
	
	wait_queue_head_t r_wait;			/* 读等待队列头 */
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

#if 0
	/* 等待事件 */
	wait_event_interruptible(dev->r_wait, atomic_read(&dev->keyRelease));	/* 等待按键值有效 */

	// NOTE:wait_event 不可以被信号打断，使用 kill -9 PID 无法杀掉任务
	// wait_event(dev->r_wait, atomic_read(&dev->keyRelease));		/* 等待按键值有效 */
#endif
	/* 等待队列项 */
	DECLARE_WAITQUEUE(wait, current);		/* 定义一个等待队列项 */

	add_wait_queue(&dev->r_wait, &wait);		/* 将等待队列项添加到等待队列头中 */
	__set_current_state(TASK_INTERRUPTIBLE);	/* 设置当前进程为可以被打断的状态 */
	schedule();									/* 切换 */

	/* 唤醒后，进程从这里运行 */
	if (signal_pending(current))	/* 判断当前进程是否有信号需要处理，返回值不为零表示有信号需要进行处理 */
	{
		ret = -ERESTARTSYS;
		goto data_err;
	}
	
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

data_err:
	__set_current_state(TASK_RUNNING);				/* 设置当前任务为运行状态 */
	remove_wait_queue(&dev->r_wait, &wait);			/* 将对应的队列项从等待队列头删除 */
	return ret;
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

	/* 唤醒进程 */
	if(atomic_read(&dev->keyRelease))
	{
		wake_up(&dev->r_wait);
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

	/* 初始化等待队列头 */
	init_waitqueue_head(&keyirq.r_wait);

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

### 2.2 非阻塞式访问

1. 驱动程序

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
#include <linux/poll.h>
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
	
	wait_queue_head_t r_wait;			/* 读等待队列头 */
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

	if(filp->f_flags & O_NONBLOCK) 
	{
		/* 非阻塞方式访问 */
		if(atomic_read(&dev->keyRelease) == 0)
		{
			return -EAGAIN;
		}
	}
	else
	{
		/* 阻塞访问方式 */
		
		/* 等待事件 */
		wait_event_interruptible(dev->r_wait, atomic_read(&dev->keyRelease));/* 等待按键值有效 */

		// NOTE:wait_event 不可以被信号打断，使用 kill -9 PID 无法杀掉任务
		// wait_event(dev->r_wait, atomic_read(&dev->keyRelease));		/* 等待按键值有效 */

		#if 0
		DECLARE_WAITQUEUE(wait, current);			/* 定义一个等待队列项 */
		add_wait_queue(&dev->r_wait, &wait);		/* 将等待队列项添加到等待队列头中 */
		__set_current_state(TASK_INTERRUPTIBLE);	/* 设置当前进程为可以被打断的状态 */
		schedule();									/* 切换 */

		/* 唤醒后，进程从这里运行 */
		if (signal_pending(current))	/* 判断当前进程是否有信号需要处理，返回值不为零表示有信号需要进行处理 */
		{
			ret = -ERESTARTSYS;
			goto data_err;
		}
		#endif
	}
	
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

	
data_err:
	#if 0
	__set_current_state(TASK_RUNNING);				/* 设置当前任务为运行状态 */
	remove_wait_queue(&dev->r_wait, &wait);			/* 将对应的队列项从等待队列头删除 */
	#endif
	return ret;
}

static unsigned int keyirq_poll(struct file *filp, struct poll_table_struct *wait)
{
	int mask = 0;
	struct keyirq_dev *dev = filp->private_data;

	poll_wait(filp, &dev->r_wait, wait);

	/* 是否可读 */
	if (atomic_read(&dev->keyRelease))
	{
		/* 按键按下，可读 */
		mask = POLL_IN | POLLRDNORM;	/* 返回POLL_IN */
	}

	return mask;
}

/* 操作集 */
struct file_operations keyirq_fops = {
	.owner 		= THIS_MODULE,
	.open 		= keyirq_open,
	.release 	= keyirq_release,
	.write 		= keyirq_write,
	.read 		= keyirq_read,
	.poll 		= keyirq_poll, 
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

	/* 唤醒进程 */
	if(atomic_read(&dev->keyRelease))
	{
		wake_up(&dev->r_wait);
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

	/* 初始化等待队列头 */
	init_waitqueue_head(&keyirq.r_wait);

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

2. 测试APP程序

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/ioctl.h>
#include <sys/select.h>
#include <sys/time.h>
#include <sys/types.h>
#include <poll.h>
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
	/* 使用select */
	// fd_set readfds;					/*  */
	// struct timeval timeout;			/* 超时时间 */

	/* 使用poll */
	struct pollfd fds;

/*
	if (argc != 3)
	{
		printf("输入错误\r\n");
	}
*/

	filename = argv[1];

	fd = open(filename, O_RDWR | O_NONBLOCK);		/* 非阻塞方式打开 */
	if(fd < 0) {
		printf("Error: Open %s fail.\r\n",filename);
		return -1;
	}

	/* 循环读取 */
	while (1)
	{	
		#if 0
		/* 使用select */
		FD_ZERO(&readfds);
		FD_SET(fd, &readfds);
		
		timeout.tv_sec = 0;
		timeout.tv_usec = 500000;		/* 超时时间500毫秒 */
		ret = select(fd + 1, &readfds, NULL, NULL, &timeout);
		switch (ret)
		{
			case 0:		/* 超时 */
				printf("select 超时\r\n");
				break;
			case -1:	/* 错误 */
				break;
			default:	/* 可以读取数据 */
				if(FD_ISSET(fd, &readfds))
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
				break;
		}
		#endif

		/* 使用poll */
		fds.fd = fd;
		fds.events = POLLIN;
		ret = poll(&fds, 1, 500);		/* 超时时间500ms */
		
		if (ret == 0)
		{
			/* 超时 */
			printf("poll 超时\r\n");
		}
		else if (ret < 0)
		{
			/* 错误 */
		}
		else
		{
			/* 可以读取 */
			if (fds.revents | POLLIN)
			{
				/* 可读取 */
				ret = read(fd, &data, sizeof(data));
				if (ret < 0)
				{ 
					
				}
				else
				{
					printf("keyValue = %#x\r\n", data);
				}
			}
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

