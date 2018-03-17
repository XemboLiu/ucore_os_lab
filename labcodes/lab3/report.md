### lab3实验报告

#### 练习1：给未被映射的地址映射上物理页

这一阶段主要修改了kern/mm/vmm.c这一文件下的do_pgfault函数。首先需要判断pte的页表所对应的物理页面是否存在，如果不存在，那么创建一个页表。

否则，如果不存在，那么*ptep == 0。这时候就需要分配一个新页并将其对应的物理地址构建一个映射。

```c
if (*ptep == 0){
    if (pgdir_alloc_page(mm -> pgdir, addr, perm) == NULL){
        // Allocating Failed, goto failed.
    }
}
```

#### 练习2：补充完成基于FIFO的页面替换算法（需要编程）

这一阶段首先完善了上一个练习中的kern/mm/vmm.c中的do_pgfault函数。如果*ptep不为0且不为NULL，那么就意味着其有一个对应的物理地址，那么就需要将其读入并进行替换。具体实现如下：

```c
if (swap_init_ok){
    // 首先将对应的页换入，如果换入失败则go failed;
    // 之后将其插入到页表的对应位置。并将其设置为swappable
}
else{
    cprintf("no swap_init_ok but ptep is %x, failed\n", *ptep);
    goto failed;
}
```

之后，则对kern/mm/swap_fifo.c中的map_swappable和swap_out_victim函数进行修改，具体修改方式如下：

```c
static int _fifo_map_swappable(struct mm_struct *mm, uintptr_t addr, struct Page * page, int swap_in){
    // Neglect unnecessary codes
    // 在entry的最后插入head即可。
    list_add(head, entry);
}
static int _fifo_swap_out_victim(struct mm_struct *mm, struct Page ** ptr_page, int in_tick){
    // Neglect unnecessary codes
    // 首先将head -> prev所对应的Page删去，之后再将*ptr_page设置为这一删去的page即可
}
/* Note: FIFO算法的含义是，我们需要link第一个到来的Page，与此同时unlink队列中最早有的Page，这两个函数所实现的就是在这一部分的内容*/
```

