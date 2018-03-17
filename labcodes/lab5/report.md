### Lab5实验报告

#### 练习1: 加载应用程序并执行

这一阶段首先修改了`kern/process/proc.c`这一文件下的`load_icode`函数。

```c
static int load_icode(unsigned char *binary, size_t size){
    /* Neglect unnecessary codes
    我们的任务是，要正确设置好trapframe中的相关内容。
    由于需要加载的是用户态进程，故
    tf -> tf_cs = USER_CS;
    tf -> tf_ds = tf -> tf_es = tf -> tf_ss = USER_DS;
    此外，需要讲tf_esp设置为user stack的顶，即
    tf -> tf_esp = USTACKTOP;
    而tf_eip需要设置为程序的进入点，即
    tf -> tf_eip = elf -> e_entry;
    之后，eflags应当设置为允许中断，即
    tf -> tf_eflags |= FL_IF;
    */
}
```

##### 请描述当创建了一个用户进程并加载了应用程序后，CPU是如何让这个应用程序最终在用户态执行起来的。即这个用户态进程被ucore选择占用CPU执行到具体执行应用程序第一条指令的整个经过。

首先，给用户进程设置用户栈，用户栈的大小为1MB。之后，将mm -> pgdir赋值到cr3寄存器中，即更新了用户进程的虚拟内存空间。最后，清空进程的中断帧，再重新设置进程的中断帧，使得在执行iret之后，CPU能够转移到用户态的特权级，并回到用户态内存空间。

最后，执行iret之后，即切换到了用户态，并从对应的位置开始执行。

#### 练习2: 父进程复制自己的空间给子进程

这一阶段主要实现了`kern/mm/pmm.c`中的`copy_range`函数。

```c
int copy_range(pde_t *to, pde_t *from, uintptr_t start, uintptr_t end, bool share){
    /* Neglect unnecessary codes
    为了将page的内容复制到npage中，首先，我们需要通过page2kva找到page和npage所对应的Kernel Virtual Address,即
    uintptr_t src_kvaddr = page2kva(page);
    uintptr_t dsa_kvaddr = page2kva(npage);
    之后就可以直接进行memcpy操作。
    memcpy(dst_kvaddr, src_kvaddr, PGSIZE);
    之后建立npage的线性映射到物理映射的转换，即
    *nptep = (page2p2(npage) & 0xfffff000) | PTE_P | PTE_W | PTE_U;
    */
}
```

##### 请简述如何设计实现Copy on Write机制

1. `fork`时的代码需要调整，`copy_range`不需要真正的执行内存复制操作，但与此同时，被“复制”的页表项需要做出一定的调整，即应当设置为只读的页面，这样，用户一旦试图修改这些页面，将会引发错误。
2. 也错误的处理代码需要进行调整。异常除去内存未分配、数据位于交换区之外，还需要添加页面的CoW内容被修改，此时应当创建一个新的页面并进行修改。

####练习3: 阅读分析源代码，理解进程执行`fork/exec/wait/exit`的实现，以及系统调用的实现。

##### 1. 请分析`fork/exec/wait/exit`是如何影响进程的执行状态的？

`fork`产生了新的可以执行的进程，其大部分内容均为复制原进程所得到的唯一的区别就在于一个进程fork的返回值是0，另一个则是1，这也导致我们可以利用这个区别使用它做完全不同的事情。

`exec`将当前进程所持有的资源释放，然后载入一个ELF文件，分配资源，为其执行创造条件，可以认为其与fork相同，同样创建了新的可以执行的进程。

`wait`等待子进程退出，获得其返回值并回收其所占用的资源，会将当前进程变为等待状态。

`exit`被调用时，回收其所持有的资源，保存其返回值，并且唤醒其父进程。并将当前进程变为假死状态（这是因为，需要保存返回值供父进程调用）

##### 2. 请给出ucore中一个用户态进程的执行状态生命周期图

这在`kern/process/proc.c`的30～40行中已经给出。

```
process state changing:
                                            
  alloc_proc                                 RUNNING
      +                                   +--<----<--+
      +                                   + proc_run +
      V                                   +-->---->--+ 
PROC_UNINIT -- proc_init/wakeup_proc --> PROC_RUNNABLE -- try_free_pages/do_wait/do_sleep --> PROC_SLEEPING --
                                           A      +                                                           +
                                           |      +--- do_exit --> PROC_ZOMBIE                                +
                                           +                                                                  + 
                                           -----------------------wakeup_proc----------------------------------
```