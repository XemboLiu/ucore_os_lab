### Lab2实验报告

#### 练习1：实现 First-fit连续物理内存分配算法

这一练习主要修改了kern/mm/default_pmm.c这一文件。需要修改文件中的四个函数：default_init, default_init_memmap, default_alloc_pages, default_free_pages。

对于default_init()这一函数，我直接采用了实验框架所提供的函数，并未对齐进行任何修改。

对于default_init_memmap(struct Page *base, size_t n)这一函数，其作用是对一段空间进行初始化。对其内容介绍如下：

```c
static void default_init_memmap(struct Page *base, size_t n)
/*两个参数的含义分别是：第一个函数base代表第一个分配的Page所在的地址，n代表所分配的页数。而函数的含义是对所分配的空间进行初始化。结合页的定义，其初始化的方法为对页中的每一个项进行初始化工作，初始化的具体方法为：
1. 将flags和property两个参数设为0。
2. 调用SetPageProperty()进行初始化。
3. 调用set_page_ref()函数对其ref进行设置，由于初始状态下没有引用，故应设为0。
4. 将这一页挂到free_list之前，调用框架所提供的list_add_before()接口。
5. 在对每一页初始化完成之后，修改nr_free和base -> property。
*/
```

对于default_alloc_pages()这一函数，其作用是分配页，具体而言，其内容如下：

```c
static struct Page* default_alloc_pages(size_t n)
/*这个函数有一个参数和一个返回值，参数的含义是，所期待分配空间的大小，而返回的是一个page，代表所分配到的空间的第一个页。需要特别说明的是，如果分配失败，则返回值为NULL。
首先，通过一个while循环来对list进行遍历。如果找到一个property >= n的区域，那么就分配这一区域。具体方法为：对区域中的每个页（通过list_next依次访问），依次调用SetPageProperty()和ClearPageProperty()对其进行初始化操作，并将其从free_list上移除。
*/
```

对于default_free_pages()这一函数，其作用是回收使用完毕的页，其具体内容如下：

```c
static void default_free_pages(struct Page *base, size_t n)
/*这个函数有两个参数，表示要free从base开始的n个页面。
首先，在free_list中找到第一个恰好大于base的位置，设之为p，因为需要将页插入在这个位置。在找到应当插入的位置之后，将这n个page依次插入到free_list当中。并对base进行设置操作(flags = 0, set_page_ref, ClearPageProperty(), SetPageProperty(), property = n)。如果插入的位置正好与p可以连接在一起，那么就将其property += p -> property，并将 p -> property 设置为0。之后对于位于其之前的页进行一次相同的操作即可。
*/
```

具体代码的实现参见/kern/mm/default_pmm.c这一文件。

#####如何优化这一算法？

这一算法在每次加入free_link的时候将所有的页都加入了这一链表，但是不论在分配还是在初始化的过程里，都仅有每一块的第一个页起到了作用。因此可以仅将每个空闲块的第一个页加入到链表中。

#### 练习2:实现寻找虚拟地址对应的页表项

补全/kern/mm/pmm.c文件中的get_pte函数。

```c
pte_t * get_pte(pde_t *pgdir, uintptr_t la, bool create)
/* 首先通过&pgdir[PDX(la)]找到对应的页表项。之后检查它是否为空 & 页表项是否为present。之后检查它是否被创建，如果没有被创建则分配一个新的页。之后创建页的地址。之后将其reference设置为1.之后获取其地址，并将这一地址对应的页进行初始化。
*/
```

#### 练习3:释放某虚拟地址所在的页并取消对应二级页表项的映射

补全/kern/mm/pmm.c文件中的page_remove_pte 函数。

```c
static inline void page_remove_pte(pde_t *pgdir, uintptr_t la, pte_t *ptep)
/* 首先要检测ptep是否存在，以及其是否为present。之后如果其ref为0则调用free_page()函数将其free。之后将ptep设置为0（即改为不存在），并invalidate对应的tlb项。
```



