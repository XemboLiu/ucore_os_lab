### Lab7 实验报告

#### 1. 理解内核级信号量的实现和基于内核级信号量的哲学家就餐问题

##### 请在实验报告中给出内核级信号量的设计描述，并说其大致执行流程

内核级信号量的实现主要包括一个结构体和四个函数

```c
typedef struct {
    int value;
    wait_queue_t wait_queue;
} semaphore_t;
void sem_init(semaphore_t *sem, int value);
void up(semaphore_t *sem);
void down(semaphore_t *sem);
bool try_down(semaphore_t *sem);
```

- 结构体中`value`表示资源量，`wait_queue`是等待资源的等待队列。
- `sem_init()`方法主要用来初始化信号量结构体
- `up()`指V操作，释放一个资源，如果有在等待的进程则将其唤醒
- `down()` 指P操作，申请一个资源，如果申请不成功则将进程放入等待队列。
- `try_down()`如果`value>0`则返回`value—`并返回成功。

##### 用户态信号量机制的设计方案

我认为，用户态信号量机制可以重用这一结构体，但需要增加三个函数，分别是`sem_init()`、`up()`和`down()`

##### 请在实验报告中给出用户态进程/线程提供信号量机制的设计方案，并比较说明给内核级提供信号量机制的异同

设计方案是，如果用户态进程down执行时，其必然会通过执行系统调用来访问一个信号量，因此可以通过信号量是否有空闲决定系统调用是否返回到原来的进程中。

#### 2. 完成内核级条件变量和基于内核级条件变量的哲学家就餐问题

这一阶段主要修改了`kern/mm/sync`下的`kern_sync.c`和`monitor.c`文件。

##### 请在实验报告中给出内核级条件变量的设计概述，并说明其大致的执行流程

ucore当中的条件变量属于前述的内核级信号量来实现，其结构包含一个信号量，一个表示其上等待的线程个数，和一个指向线程所属的管程的指针。

wait操作：将Count加一，如果管程中有正在等待的线程，就将其唤醒。否则释放互斥锁，最后对条件变量的信号量执行`down()`操作，将自己加入等待队列。再次运行时，将count减一并退出。

signal操作：如果count不大于0，则什么也不做。否则，对条件变量的信号量进行`up()`操作唤醒其上等待的线程。再次运行时，释放信号量`monitor -> next`并退出。

##### 内核级条件变量的设计

内核级条件变量主要包括一个结构体和两个函数

```c
typedef struct condvar{
    semaphore_t *sem;
    int count;
    monitor_t *owner;
} condvar_t;

void cond_signal(condvar_t *cvp);
void cond_wait(condvar_t *cvp);
```

- `sem`是信号量，此处借用信号量来实现条件变量，等待队列在`sem`中
- `count`记录了等待队列的长度
- `owner`记录了这一条件变量所属的管程
- `cond_signal()`函数用于唤醒正在等待该条件变量的进程，并将自己加入管程的`next`
- `cond_wait()`函数用于等待某个条件变量，先将条件变量等待队列的长度+1，若没有进程等待该条件，则相当于空操作。

##### 内核级管程的实现

```c
typedef struct monitor{
    semaphore_t mutex;
    semaphore_t next;
    int next_count;
    condvar_t *cv;
} monitor_t;

void monitor_init(monitor_t *cvp, size_t num_cv);
```

- `mutex`用于控制执行管程的进程只有一个
- `next`是管程中用于记录因为执行`signal`而进入等待队列的进程
- `next_count`是用于记录`next`等待队列的进程数
- `cv`是条件变量的数组
- `monitor_init()`用于初始化管程

######以lab7中相关内容的实现为例，介绍一下如何使用管程

```c
	down(&(mtp -> mutex));
	/* Begin
	Neglect unnecessary codes
	end */
	if (mtp -> next_count > 0)
        up(&(mtp -> next));
	else
        up(&(mtp -> mutex))
```

##### 用户态条件变量机制设计

与前述用户态信号量的机制类似，可以重用`condvar`结构体，但是需要在系统调用中添加对应的`cond_signal()`和`cond_wait()`接口。

##### 能否不用基于信号量机制来完成条件变量？

我认为是可以的。只需要将PV操作更换为基于管程的实现。

但是在实际的使用中，我认为这种做法会降低效率。