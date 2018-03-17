### Lab6 实验报告

#### 1. 使用Round Robin调度算法

Round Robin算法的实现方式是：在一开始，每个进程被分配一个固定数目的时间片，每次时钟中断进程的可用时间片数目减1，当最终减到0时，就表明该进程的时间片已经耗尽，需要切换到下一个进程的执行。在uCore中，进程调度是通过一个链表来实现的。每次总是选择在开头的进程运行，而时间片耗尽之后，该进程被从队列头部移除，并插入到队尾。此外，需要讲resched标记为置为1。

对sched_class函数中各个函数指针的解释如下：

- init：初始化调度程序
- enqueue：将一个可以运行的进程加入到调度器之中。
- dequeue：进程出队，将一个进程移除。
- pick_next：在之前的进程的时间片用尽后，选择下一个需要运行的进程。
- proc_tick：用于时钟中断的处理。

##### 请在实验报告中简要说明如何实现“多级反馈队列调度算法”，给出概要设计。

多级反馈队列调度算法，实际上是可以基于uCore的Round Robin算法进行实现。大致的实现方法为：使用多个队列而非一个队列。对于这多个队列而言，每个队列中的进程都具有不同的优先级。此外，操作系统会给每个不同优先级进程一个时间片：优先级高的进程时间片短。对于每一个进程，都有一个固定的执行次数，如果在达到该执行次数之后，进程仍然没有结束，则将其放到下一个优先级所对应的队列中。

#### 2. 实现Stride Scheduling调度算法

这一阶段主要修改了`kern/schedule/default_sched_stride_c`文件并使用其覆盖了该文件夹的`default_sched.c`这一文件。对四个需要修改的函数解释如下

```c
static void stride_init(struct run_quene* rq){
    /*这一个函数的作用是初始化stride，具体而言，初始化的过程可以分为三步：第一步，初始化就绪进程的列表。此外，由于初始化时没有进程，故将 rq -> lab6_run_pool 和 rq -> proc_num 分别赋值为NULL和0。
    */
}
// 对于之后的函数，框架中允许我们选择采用skew heap或者是list，而我所选用的数据结构为skew_heap,特此注明。
static void stride_enqueue(struct run_queue* rq, struct proc_struct *proc){
    /*这一函数的作用是，讲一个进程proc插入到运行队列rq中。
    首先调用skew_heap_insert()函数，将proc插入到rq -> lab6_run_pool中。
    在这之后，将process的rq绑定为rq。并将其time_slice设置为rq的max_time_slice。
    最后，将rq的proc_num + 1（这是因为，有一个新的进程进入了队列中）
    */
}
static void stride_dequeue(struct run_queue *rq, struct proc_struct *proc){
    /*这一函数的作用是，将proc所指定的进程从rq中删除。
    与插入类似，首先调用shew_heap_remove()函数，将proc从rq -> lab6_run_pool中删除。之后将rq -> proc_num - 1（这是因为进程被从队列中删去）。
    */
}
static void stride_pick_next(struct run_queue *rq){
    /*pick_next函数的含义为，从rq中选择接下来要执行的进程。由于采用了skew_heap故可以直接获取堆顶的元素。这一步，首先调用le2proc来获取进程。之后依据进程的优先级设定不同的proc -> lab6_stride。这一成员函数表示运行的步长。这里我采用的设置方法为：如果lab6_priority是0（这时，除法的商不存在），那么就将lab6_stride加上BIG_STRIDE。否则，如果lab6_priority不为0，就将其设置为BIG_STRIDE除以priority所得到的结果。
    需要说明的是，这里BIG_STRIDE需要自行设定。我设置的为0x112810，设置为这一结果是因为我的学号是2015011281。
    */
}
static void stride_proc_tick(struct run_queue *rq, struct proc_struct *proc){
    /*这一函数的在第一问中已经说明，故这里不再赘述。
    */
    proc -> time_slice--;
    if (proc -> time_slice == 0)
        proc -> need_resched = 1;
}
```

