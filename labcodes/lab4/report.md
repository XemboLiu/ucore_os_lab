### Lab4实验报告

#### 练习1: 分配并初始化一个进程控制块

这一阶段主要修改了`kern/process/proc.c`中的`alloc_proc`函数。其主要用处是完成`proc_struct`结构的初始化。

由代码中和gitbooks中的提示，我们知道，其需要初始化的变量有`state / pid/ runs/ kstack/ need_resched/ parent/ mm/ context/ tf/ cr3/ flags/ name`几项

```c
static struct proc_struct* alloc_proc(void){
    /* Neglect unnecessary codes
    完成对 proc_struct 中成员变量的初始化
    proc -> state: 进程需要被初始化，因此这里要置为PROC_UNINIT
    proc -> pid: 由于没有process ID, 故这里需要置为0
    proc -> runs: 进程的running time，由于刚被初始化，故被置为0
    proc -> kstack: Process Kernel Stack，由于未被分配，置为0
    proc -> need_resched: 由于进程没有实际的任务，所以应该为false(0)
    proc -> parent: 空进程没有父进程，故为NULL
    proc -> mm: 空进程没有mm，故置为NULL
    proc -> trapframe: 由于是空进程，故为NULL
    proc -> cr3: 页表位置被置为boot的页表位置，即boot_cr3
    proc -> flags: 进程的flags，由于是空进程，置为NULL
    proc -> name: 由于是空进程，故没有名字。考虑到命名规律，实际的代码为：proc -> name[0] = '\0';
    */
}

```

##### 请说明`proc_struct`中`struct context context`和`struct trapframe *tf`成员变量的含义和其在本实验中的作用

`context`指的是进程的“上下文”，其主要目的是保存了进程运行过程中的寄存器（恢复进程所必须的寄存器的值）。需要特别说明的是，其中没有保存%eax，这是因为%eax在fork时本身就是用作返回值的。

`tf`是指进程内核栈中的一个地址，指向该进程的陷入帧，在进程切换返回之后，系统可以根据这个陷入帧回到进程之前的状态。

#### 练习2: 为新创建的内核线程分配资源

这一阶段主要修改了`kern/process/proc.c`中的`do_fork`函数。实现过程如下：

```c
int do_fork(uint32_t clone_flags, uintptr_t stack, struct trapframe *tf){
    /* Neglect unnecessary codes
    首先，调用proc = alloc_proc(); 来分配一个新的proc_struct。
    之后判断proc是否为NULL，若为NULL则跳转到bad_fork_cleanup_proc。
    之后，设置proc的pid和parent。pid需要通过get_pid()函数来实现设置，而parent则为当前的进程(proc -> parent = current;)
    如果setup_kstack(proc) != 0则说明出现了错误，跳转到bad_fork_cleanup_proc
    之后依次调用copy_mm和copy_thread，分别复制其共享的mm和其context值。
    之后将proc插入到hash_list和proc_list中。分别调用list_add和hash_proc函数进行设置。
    在设定完成之后，进程就可以被激活了。故调用wakeup_proc(proc)来完成这一过程。
    最后，使用ret = proc -> pid。这是因为这一函数的返回值应为子进程的进程号。
    */
}
```

##### UCore是否做到给每个新fork的线程一个唯一的id？请说明你的分析和理由

UCore的进程是唯一的，分配进程id的任务如上述，是通过调用get_pid()函数来实现的。而get_pid的具体做法为：每次遇到一个新的进程id则加1，如果进程好用尽则从1开始循环。get_pid会自动检查已经分配的进程序号和执行完的进程好避免重复。

#### 练习3: 阅读代码，理解proc_run函数和它调用的函数是如何完成进程切换的

`proc_run()`函数将当前运行的进程从全局变量`current`变为传入的参数`proc`。函数先后完成了以下的工作：将`current`置为`proc`，切换内核堆栈，载入新进程的页表，最后调用`switch_to()`完成进程的切换。ß

