---
title: IMX6ULL嵌入式Linux驱动学习笔记（七）
date: 2020-11-17 22:45:49
tags: [嵌入式, IMX6ULL, 学习笔记]
keywords:
description: 正点原子alpha开发板IMX6ULL嵌入式Linux开发学习笔记。
---

**IMX6ULL嵌入式Linux驱动开发学习**

以下内容是我在学习正点原子`IMX6ULL`开发板`alpha`中记录的笔记，部分摘录自正点原子`IMX6ULL开发手册`。

# Linux并发与竞争

## 一、并发与竞争

多线程对共享资源同时进行访问，比如全局变量，就会产生并发与竞争现象。以打印机为例，当线程A和线程B同时操作打印机时，就会出现竞争现象，如果没有处理就会导致数据错乱。

<!--more-->

## 二、原子操作`atomic`  

对整形变量或者位进行保护，确保对其进行操作时是最小操作，不会被干扰。

```c
typedef struct {
    int counter;
} atomic_t;

atomic_t a = ATOMIC_INIT(0);	// 定义原子变量a并赋初值为0；
```

原子操作`API`函数

```c
AIOMIC_INIT(int i)							// 定义原子变量的时候对其初始化
int atomic_read(atomic_t* v)				// 读取原子变量的值，并返回
void atomic_set(atomic_t* v, int i)			// 给原子变量设置为i值
void atomic_add(int i, atomic_t* v)			// 给原子变量加上i值
void atomic_sub(int i, atomic_t* v)			// 给原子变量减去i值
void atomic_inc(atomic_t* v)				// 原子变量自增1
void atomic_dec(atomic_t* v)				// 原子变量自减1
int atomic_dec_return(atomic_t* v)			// 原子变量减1，并且返回v的值
int atomic_inc_return(atomic_t* v)			// 原子变量加1，并且返回v的值
int atomic_sub_and_test(int i, atomic_t* v)	// 从v减i，如果结果为0就返回真，否则返回假
int atomic_add_and_test(int i, atomic_t* v)	// 给v加i，如果结果为0就返回真，否则返回假
int atomic_dec_and_test(atomic_t* v)		// 从v减1，如果结果为0就返回真，否则返回假
int atomic_inc_and_test(atomic_t* v)		// 给v加1，如果结果为0就返回真，否则返回假
```

## 三、原子位操作

原子位操作不像原子整形变量那样有个`atomic_t`的数据结构，原子位操作是直接对内存进行操作。

`API`函数如下

```c
void set_bit(int nr, void* p)				// 将p地址的第nr位置1
void clear_bit(int nr, void* p)				// 将p地址的第nr位清零
void change_bit(int nr, void* p)			// 将p地址的第nr位翻转
int test_bit(int nr, void* p)				// 获取p地址的第nr位的值
int test_and_set_bit(int nr, void* p)		// 将p地址的第nr位置1，并返回nr位原来的值
int test_and_clear_bit(int nr, void* p)		// 将p地址的第nr位清零，并返回nr位原来的值
int test_and_change_bit(int nr, void* p)	// 将p地址的第nr位翻转，并返回nr位原来的值
```

## 四、自旋锁`spinlock`

- 用于多核`SMP` 。
- 适合短时间加锁，轻量级加锁。  
- 自旋锁会自动禁止抢占。
- 使用自旋锁，要注意死锁现象的发生，被自旋锁保护的临界区一定不能调用任何能够引起睡眠和阻塞的`API函数`。（线程与线程中之间，线程与中断之间）。

`API`函数

```c
spinlock_t lock;						// 定义自旋锁
DEFINE_SPINLOCK(spinlock_t lock)		// 定义并初始化一个自旋锁变量
int spin_lock_init(spinlock_t* lock)	// 初始化自旋锁
void spin_lock(spinlock_t* lock)		// 获取指定的自旋锁，也叫加锁
void spin_unlock(spinlock_t* lock)		// 是否指定的自旋锁
int spin_trylock(spinlock_t* lock)		// 尝试获取指定的自旋锁，如果没有获取到就返回0
int spin_is_locked(spinlock_t* lock)	// 检查指定的自旋锁是否被获取，如果没有被获取就返回非0，否则返回0

void spin_lock_irq(spinlock_t* lock)	// 禁止本地中断，并获取自旋锁。
void spin_unlock_irq(spinlock_t* lock)	// 激活本地中断，并释放自旋锁。
void spin_lock_irqsave(spinlock_t* lock, unsigned long flags)	// 保存中断状态，禁止本地中断，并获取自旋锁。
void spin_unlock_irqrestore(spinlock_t* lock, unsigned long flags)	// 将中断状态恢复到以前的状态，并且激活本地中断，释放自旋锁。
```

使用示例：

```c
DEFINE_SPINLOCK(lock) 						// 定义并初始化一个锁

/* 线程 A */
void functionA ()
{
    unsigned long flags;					// 中断状态
    spin_lock_irqsave(&lock, flags)			// 获取锁
    /* 临界区 */
    spin_unlock_irqrestore(&lock, flags)	// 释放锁
}

/* 中断服务函数 */
void irq() 
{
    spin_lock(&lock)		// 获取锁
    /* 临界区 */
    spin_unlock(&lock)		// 释放锁
}
```

## 五、信号量`semaphore`

* 信号量可以使等待资源线程进入休眠状态，因此适用于占用资源比较久的场合。  
* 信号量不能用于中断中，因为信号量会引起休眠，中断不能休眠。
* 信号量会将等待信号量中休眠的线程唤醒。  
* 如果共享资源的持有时间比较短，那就不适合适用信号量了。

`API`函数

```c
DEFINE_SEAMPHORE(name)							// 定义一个信号量，并且设置信号量的值为1
void sema_init(struct semaphore* sem, int val)	// 初始化信号量sem，设置信号量值为val
void down(struct semaphore* sem)				// 获取信号量，因为会导致休眠，因此不能在中断中使用。
int down_trylock(struct semaphore* sem)			// 尝试获取信号量，如果能获取到信号量就获取，并且返回0。如果不能就返回非0，并且不会进入休眠。
int down_interruptible(struct semaphore* sem)	// 获取信号量，和down类似，只是使用down进入休眠状态的咸亨不能被信号打断。而使用此函数进入休眠以后是可以被信号打断的。
void up(struct semaphore* sem)					// 释放信号量
```

使用方式如下：

 ```c
struct semaphore sem;		// 定义信号量
sema_init(&sem, 1);			// 初始化信号量

down(&sem);					// 申请信号量
/* 临界区 */
up(&sem);					// 释放信号量
 ```

## 六、互斥锁`mutex`

- 互斥体可以导致休眠，因此不能在中断中使用，中断中只能使用自旋锁。  
- 互斥体保护的临界区可以调用引起阻塞的`API函数` 。
- 一次只有一个线程可以持有互斥体，因此必须由`mutex`的持有者释放`mutex`。不能递归上锁和解锁。

`API`函数：

```c
DEFINE_MUTEX(name)						// 定义并初始化一个 mutex 变量。
void mutex_init(struct mutex* lock)		// 初始化 mutex。
void mutex_lock(struct mutex* lock)		// 获取 mutex，也就是给 mutex 上锁。如果获取不到就进休眠。
void mutex_unlock(struct mutex* lock)	// 释放 mutex，也就给 mutex 解锁。
int mutex_trylock(struct mutex* lock)	// 尝试获取 mutex，如果成功就返回 1，如果失败就返回 0。
int mutex_is_locked(struct mutex* lock)	// 判断 mutex 是否被获取，如果是的话就返回1，否则返回 0。
int mutex_lock_interruptible(struct mutex* lock)	// 使用此函数获取信号量失败进入休眠以后可以被信号打断
```

使用示例：

```c
struct mutex lock;			// 定义一个互斥体
mutex_init(&lock);			// 初始化互斥体

mutex_lock(&lock);			// 上锁
/* 临界区 */
mutex_unlock(&lock);		// 解锁
```
