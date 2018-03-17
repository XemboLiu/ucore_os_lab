### Lab6 实验报告

#### 1. 完成读文件操作的实现

这一阶段首先修改了`sfs_inode.c`中`sfs_io_nolock`读文件中数据的实现代码。

```c
static int sfs_io_nolock(struct sfs_fs *sfs, struct sfs_inode *sin, void *buf, off_t offset, size_t *alenp, bool write) {
    /* Neglect unnecessary codes
    我们的实验代码所处理的，主要是开头和结尾地址没有对齐的问题。
    首先，通过sfs_bmap_load_nolock来读取节点的值。
    再通过sfs_buf_op将数据都加入到缓冲区当中。
    */
}
```

##### 请在实验报告中给出设计实现"UNIX的PIPE机制"的概要设计方案，鼓励给出详细设计方案

首先，我们需要两个函数，分别是管道的读函数和管道的写函数（不妨命名为`pipeRead()`和`pipeWrite()`）。管道读函数将复制物理内存中的字节以读出数据，而管道写函数通过将字节复制到VFS索引节点指向的物理内存而写入数据。为了实现内核同步对管道的访问。内核使用了锁、等待队列和信号。当写进程向管道中写入的时候，利用的是标准的库函数`write()`，系统根据库函数传递的文件描述符，可以找到该文件的file结构。file结构中制定了用来进行写操作的函数地址。之后，内核利用该函数来完成写操作。写入函数在向内存中写入数据之前，必须首先检查VFS索引节点中的信息，同时满足如下条件时，才能进行实际内存的肤质操作。

- 内存中有足够的空间可以容纳所有要写入的数据
- 内存没有被读取程序锁定

如果同时满足上述条件，则写入函数将首先锁定内存，然后从进程的地址空间中将数据复制到内存。之后，内核将调用调度程序，而调度程序将调度其他进程运行。写入进程实际处于可中断的等待状态，当内存中有足够的空间可以容纳写入数据，或者内存被解锁时，读取进程会唤醒写入进程。这时，写入进程将接收到信号。当数据写入内存之后，内存被解锁，而所有休眠在索引节点的读取进程会被唤醒。管道的读取和写入过程类似，但是进程可以休眠在索引节点的等待队列中等待要写入的进程写入数据，当所有的进程完成了管道操作之后，管道的索引节点被丢弃，而共享数据页也被释放。

#### 2. 完成基于文件系统的执行程序机制的实现

这一阶段主要修改了`kern/process/proc.c`中的`load_inode`函数。具体实现如下

```c
static int load_inode(int fd, int argc, char** kargv){
    /*
    这一版本的load_inode和之前版本的load_inode并没有较大的区别，但是一个是从内存中读取elf映像，而另一个是要通过文件系统的相关操作来读取elf映像。读取的过程包括以下几个方面的内容：首先，需要读取elf文件的头部并且进行最基本的合法性检查。接下来，一次加载elf中的每个代码段并为之分配内存，设置权限，然后配置好应用程序的运行栈，最后修改trapframe中相关寄存器的值，以使得context的相关切换可以顺利的进行。
    */
}
```

##### 设计实现基于“UNIX的硬链接和软链接机制”的概要设计方案

- 硬连接：新建文件指向同一个inode，在inode中增加引用计数，删除文件时引用-1，减为0之后释放inode和block。
- 软链接：在inode和block信息中增加一个bool值区分起是否是软链接，软链接有独立的文件但其中记录的是链接的文件路径。读写文件时根据软链接block中记录的内容进行文件的查找工作。