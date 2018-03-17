###Lab1实验报告

####练习1

#####1.1 操作系统镜像文件ucore.img是如何一步一步生成的？（需要比较详细的解释Makefile中每一条相关命令和命令参数

生成ucore.img的相关代码为：

```makefile
$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
```

因此，在生成ucore.img之前，需要先生成bootblock、kernel。

生成bootblock的相关代码为：

```makefile
$(bootblock): $(call toobj, $(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7c00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
```

可以看到，要生成bootblock，首先需要生成“boot/文件夹下所有.S, .c文件对应的.o文件”和"sign"，即需要生成

bootasm.o, bootmain.o和sign

生成bootasm.o和bootmain.o文件的代码为

```makefile
bootfiles = $(call listf_cc,boot)
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))
```

而生成sign文件对应的代码为

```makefile
$(call add_files_host,tools/sign.c,sign,sign)
$(call create_target_host,sign,sign)
```

生成bootblock的过程中，首先从bootasm.o和bootmain.o生成bootblock.o，对应的代码为：

```makefile
$(V)$(LD)$(LDFLAGS) -N -e start -Ttext 0X7C00 $^ -o $(call toobj,bootblock)
```

之后采用objcopy将bootblock.o拷贝为bootblock.out,并采用sign文件处理bootblock.out，对应的代码为

```makefile
@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
```

至此bootblock即生成完毕。而生成kernel的相关代码为：

```makefile
$(kernel): tools/kernel.ld
$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $call(asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)
```

其中KOBJS为obj/kern和obj/lib下指定的文件夹。

至此ucore.img所依赖的文件全部生成完毕。接下来的操作为：

```makefile
$(V)dd if=/dev/zero of=$@ count=10000
#生成一个含有10000个块的文件
$(V)dd if=$(bootblock) of=$@ conv=notrunc
#将bootblock中的内容写到第一个块内
$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
#从第二个块开始写kernel中的内容
```

##### 一个被系统认为是复合规范的硬盘主引导扇区的特征是什么？

一个合法的主引导扇区，其结束的两个字节应为"55 AA" [Reference](https://www.cnblogs.com/CasonChan/p/4546658.html)

sign.c的相关代码也证明了这一点。

#### 练习2

#####2.1 从CPU加电后执行的第一条指令开始，单步跟踪BIOS的运行

在本次试验中，需要修改lab1/tools/gdbinit，具体方法为在结尾加入

```shell
set architecture i8086
```

并执行

```shell
make debug
```

之后使用

```shell
si
```

即可单步跟踪BIOS。

##### 2.2 在初始化位置0x7c00设置实地址断点，测试断点正常

在0x7c00设置实地址断点的方法为：在gdbinit的结尾加入

```shell
set architecture i8086
	b *0x7c00
	c
	x /2i $pc
	set architecture i386
```

并删去原有的gdbinit文件中结尾的

```shell
continue
```

行，之后运行

```shell
make debug
```

即可

##### 2.3 从0x7c00开始跟踪代码运行，将单步跟踪反汇编得到的代码与bootasm.S和bootblock.asm进行比较。

首先，需要改写makefile文件，在调用qemu时增加以下参数：

```shell
-d in_asm -D $(BINDIR) q.log
```

便可以得在bin/q.log中得到输出，具体的输出为

```
----------------
	IN: 
	0x00007c00:  cli    
	
	----------------
	IN: 
	0x00007c01:  cld    
	0x00007c02:  xor    %ax,%ax
	0x00007c04:  mov    %ax,%ds
	0x00007c06:  mov    %ax,%es
	0x00007c08:  mov    %ax,%ss
	
	----------------
	IN: 
	0x00007c0a:  in     $0x64,%al
	
	----------------
	IN: 
	0x00007c0c:  test   $0x2,%al
	0x00007c0e:  jne    0x7c0a
	
	----------------
	IN: 
	0x00007c10:  mov    $0xd1,%al
	0x00007c12:  out    %al,$0x64
	0x00007c14:  in     $0x64,%al
	0x00007c16:  test   $0x2,%al
	0x00007c18:  jne    0x7c14
	
	----------------
	IN: 
	0x00007c1a:  mov    $0xdf,%al
	0x00007c1c:  out    %al,$0x60
	0x00007c1e:  lgdtw  0x7c6c
	0x00007c23:  mov    %cr0,%eax
	0x00007c26:  or     $0x1,%eax
	0x00007c2a:  mov    %eax,%cr0
	
	----------------
	IN: 
	0x00007c2d:  ljmp   $0x8,$0x7c32
	
	----------------
	IN: 
	0x00007c32:  mov    $0x10,%ax
	0x00007c36:  mov    %eax,%ds
	
	----------------
	IN: 
	0x00007c38:  mov    %eax,%es
	
	----------------
	IN: 
	0x00007c3a:  mov    %eax,%fs
	0x00007c3c:  mov    %eax,%gs
	0x00007c3e:  mov    %eax,%ss
	
	----------------
	IN: 
	0x00007c40:  mov    $0x0,%ebp
	
	----------------
	IN: 
	0x00007c45:  mov    $0x7c00,%esp
	0x00007c4a:  call   0x7d0d
	
	----------------
	IN: 
	0x00007d0d:  push   %ebp
```

其与bootasm.S中的代码相同。

#####2.4 自己找一个bootloader或内核中的代码位置，设置断点并进行调试

修改上一问中

```makefile
b *0x7c00
```

的位置即可。

#### 练习3:分析bootloader进入保护模式的过程。（要求在报告中写出分析）

这一练习主要针对bootasm.S这一文件进入分析。

首先可以看到，在start时，首先执行了如下的代码，清空段寄存器。（需要注意的是，AT&T汇编与上学期所学的MIPS不同，赋值方向为从左到右赋值）

```assembly
.code 16
	cli					# Disable Interrupts
	cld					# String operations
	xorw %ax, %ax		# Clear segment number
	movw %ax, %ds		# -> Data segment
	movw %ax, %es		# -> Extra segment
	movw %ax, %ss		# -> Stack segment
```

之后，采用如下的代码段进行Enable A20的地址线操作。[A20的作用参见这里](http://blog.sina.com.cn/s/blog_5d66c2780100bp8n.html)

```assembly
seta20.1:
	inb $0x64, %al		# Wait for not busy
	testb $0x2, %al			
	jnz seta20.1
	movb $0xd1, %al		# 0xd1 -> port 0x64
	outb %al, $0x64		# 0xd1 means write

seta20.2:
	inb 0x64, %al
	testb $0x2, %al
	jnz seta20.2
	movb $0xdf, %al		# 0xdf -> port 0x60
	outb %al, $0x60		# 0xdf = 11011111, means set A20 bits to 1
```

之后初始化gdt表

```assembly
	lgdt gdtdesc
```

进入保护模式：通过将cr0寄存器PE位置1就开启了保护模式

```assembly
	movl %cr0, $eax
	orl $CR0_PE_ON, %eax
	movl %wax, %cr0
```

通过长跳转更新cs的基地址

```assembly
	ljmp $PORT_MODE_CSEG, $protcseg
```

设置段寄存器，并建立堆栈

```assembly
	movw $PORT_MODE_DSEG, %ax		# Our data segment selector
	movw %ax, %ds					# -> DS: Data segment
	movw %ax, %es					# -> ES: Extra segment
	movw %ax, %fs					# -> FS
	movw %ax, %gs					# -> GS
	movw %ax, %ss					# -> SS: Stack segment
	movl $0x0, %ebp
	movl $start, %esp
```

之后转到保护模式完成，进入boot主方法

```assembly
	call bootmain
```

#### 练习4:分析bootloader加载ELF格式的OS的过程

bootmain.c文件中共有日下的几个函数：

```c
// wait for disk ready
static void waitdisk(void);
// read a single sector at @secno into @dst
static void readsect(void *dst, uint32_t secno);
// read @count bytes at @offset from kernel into virtual
static void readseg(uintptr_t va, uint32_t count, uint32_t offset);
// the entry of bootloader
void bootmain(void);
```

waitdisk的作用较为简单。故不在这里进行分析。

readrect函数的作用为，从设备的secno扇区读取数据到dst位置。

```c
static void
readsect(void *dst, uint32_t secno){
	waitdisk();
	outb(0x1F2, 1);						// count = 1
	outb(0x1F3, secno & 0xFF);
	outb(0x1F4, (secno >> 8) & 0xFF);
	outb(0x1F5, (secno >> 16) & 0xFF);
	outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
	outb(0x1F7, 0x20);					// cmd 0x20 - read sectors
	// wait for disk to be ready
	waitdisk();
	// read a sector
	insl(0x1F0, dst, SECTSIZE / 4);
}
```

readseg简单对readsect进行了包装，可以从设备读取任意长度的内容。

```c
static void
readseg(uintptr_t va, uint32_t count, uint32_t offset){
	uintptr_t end_va = va + count;
	// round down to sector boundry
	va -= offset % SECTSIZE;
	// translate from bytes to sectors; kernel starts at sector 1
	uint32_t secno = (offset / SECTSIZE) + 1;
	// if this is too slow, we could read lots of sectors at a time.
	// We'd write more to memory than asked, but it doesn't matter --
	// we load in increasing order.
    for(; va < end_va; va += SECTSIZE, secno ++){
    	readsect((void*)va, secno);
    }
}
```

而bootmain函数为bootloader的入口，其作用如下：

注：由于ELF文件中描述了ELF文件所应当加载到的位置，及内核的入口位置，所以可以直接通过这些存储的信息找到内核的入口。

```c
void
bootmain(void){
	// read the 1st page off disk
	readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);
	// judge if this is a valid ELF
    if (ELFHDR -> e_magic != ELF_MAGIC){
    	goto bad;
    }
    struct proghdr *ph, *eph;
    // load each program segment(ignores ph flags)
    ph = (struct proghdr*)((uintptr_t)ELFHDR + ELFHDR -> e_phoff);
    eph = ph + ELFHDR -> e_phnum;
    for(; ph < eph; ph++){
    	readseg(ph -> p_va & 0xFFFFFF, ph -> p_memsz, ph -> p_offset);
    }
    // call the entry point from the ELF header
    // note: does not return
    ((void (*)(void))(ELFHDR -> entry & 0xFFFFFF));
bad:
	outw(0x8A00, 0x8A00);
	outw(0x8A00, 0x8A00);
	while(1);
}
```

#### 练习5: 实现函数调用堆栈跟踪函数

这里修改了kdebug.c中的print_stackframe函数，其具体的含义见对应的代码。

首先，堆栈最深一层的输出为

```shell
ebp:0x00007bf8 eip:0x00007d6e args: 0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8
	<unknow>: --0x00007d6d --
```

其对应的是第一个使用堆栈的函数，bootmain.c中的bootmain。bootloader设置的堆栈从0x7c00开始，使用"call bootmain"转入bootmain函数，call指令压栈，所以bootmain中ebp为0x7bf8。

#### 练习6:完善中断和初始化处理

对问题回答如下：

###### 1. 中断描述符表（也可简称为保护模式下的中断向量表）中一个表项占多少字节？其中哪几位代表中断处理代码的入口

中断向量表一个表项占用8字节，其中2-3字节是段选择字，0-1字节和6-7字节拼成位移，两者联合便是中断处理程序的入口地址。

###### 请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init，在idt_init函数中，依次对所有中断入口进行初始化。使用mmu.h中的SETGATE宏，填充idt数组内容，每个中断的入口由tools/vectors.c生成，使用trap.c中声明的vectors数组即可。

编程题目解答见对应文件，实现过程的说明如下：

首先通过

```c
extern uintptr_t __vectors[];
```

获得中断处理向量的位置，之后对idt中的每一项进行setgate操作，之后将T_SWITCH_TOK这一项的权限设置为用户态，之后调用lidt函数让CPU得知我们已经对相应内容的设置进行完毕。

######请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数中处理时钟中断的部分，使操作系统每遇到100次中断后，调用print_ticks子程序，向屏幕上打印一行文字"100 ticks"

编程题目解答见对应文件，实现过程较为简单，故不在这里进行说明。

#### 扩展练习1：增加syscall功能

由于流程已经在提示中进行了规范，故这里仅做简要说明：

在trap_dispatch中，将iret时会从堆栈弹出的段寄存器进行修改，对Switch to User

```c
	tf -> tf_cs = USER_CS;
	tf -> tf_ds = USER_DS;
	tf -> tf_es = USER_DS;
	tf -> tf_ss = USER_DS
```

对Switch to Kernel

```c
	tf -> tf_cs = KERNEL_CS;
	tf -> tf_ds = KERNEL_DS;
	tf -> tf_es = KERNEL_DS;
```

在lab1_switch_to_user中，处理如下：

```c
	asm volatile(
        "sub $0x8, %%esp \n"
        "int %0 \n"
        "movl %%ebp, %%esp"
        :
        : "i"(T_SWITCH_TOU)
    );
```

在lab1_switch_to_kernel中，调用T_SWITCH_TOK中断，具体实现方式如下：

```c
	asm volatile(
        "int %0 \n"
        "movl %%ebp, %%esp \n"
        :
        :"i"(T_SWITCH_TOK)
    );
```

此外，需要在trap_dispatch转到user态时，将调用io所需权限降低。

```c
	tf -> tf_eflags |= 0x3000
```

