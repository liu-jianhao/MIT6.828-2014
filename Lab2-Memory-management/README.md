# 实验二：内存管理
[实验二](https://pdos.csail.mit.edu/6.828/2014/labs/lab2/)

## 实验准备
按照指导更新了lab2的代码后，多了这几个文件：
```
inc/memlayout.h
kern/pmap.c
kern/pmap.h
kern/kclock.h
kern/kclock.c
```
+ `memlayout.h`描述了必须通过修改`pmap.c`来实现的虚拟地址空间的布局。
+ `memlayout.h`和`pmap.h`定义了`PageInfo`用于跟踪哪些物理内存页面是空闲的结构。
+ `kclock.c`和`kclock.h`操纵PC的电池供电时钟和CMOS RAM硬件，其中BIOS记录PC包含的物理内存量等。
+ `pmap.c`中的代码需要读取此设备硬件以确定有多少物理内存，但代码的这一部分是需要完成的，但不需要知道CMOS硬件如何工作的细节。
特别注意`memlayout.h`和`pmap.h`，因为本实验要求使用并理解它们包含的许多定义。`inc/mmu.h`也有用。

## 第一部分：物理页面管理
操作系统必须跟踪物理`RAM`的哪些部分是空闲的以及哪些是当前正在使用的。

`JOS`以页面粒度管理PC的物理内存，以便它可以使用`MMU`映射和保护每个分配的内存。

需要编写物理页面分配器。它通过链接的`struct PageInfo`对象列表跟踪哪些页面是空闲的（与`xv6`不同，它们不嵌入在空闲页面中），每个对应于一个物理页面。

在编写剩余的虚拟内存实现之前，需要编写物理页面分配器，因为页表管理代码需要分配用于存储页表的物理内存。

### 练习1. 在文件`kern/pmap.c`中，必须为以下函数实现代码（可能按给定的顺序）。
```
boot_alloc()
mem_init()
page_init()
page_alloc()
page_free()
```
`check_page_free_list()`和`check_page_alloc()`是用于测试的物理页面分配器。

启动`JOS`并查看`check_page_alloc()`报告是否成功。修复代码，使其通过。添加自己的`assert()`有助于验证假设是否正确。

#### Part0：理解什么是两级页表
+ 内存分页管理的基本原理是将整个主内存区域划分成`4096`字节为一页的内存页面。
+ 程序申请使用内存时，就以内存页为单位进行分配。上面提到了线性地址经过分页机制的转换变成物理地址，但是没有提到如何转换。
+ 其实是通过两个表，一个是页目录表`PDE`，也叫一级目录，另一个是二级页表`PTE`。进程的虚拟地址需要首先通过其局部段描述符变换为CPU整个线性地址空间中的地址， 然后再使用页目录表PDE（一级页表）和页表PTE（二级页表）映射到实际物理地址页上。
+ 页表中，每项的大小是32b，其中20b用来存放页面的物理地址，12b用于存放属性信息。页表含有1M个表项，每项4B。第一级表是页目录，存放在1页4k页面中，含有1K个表项。第二级是页表，也是1K个表项。如下图所示：

![]()

#### Part1：理解一些宏函数
`typeof`是`GNU C`标准中的一个扩展特性，类似于`C++11`中的`decltype`，就是自动推导表达式的数据类型

`ROUNDUP`的作用就是计算传进来的字节数所需要的页面数
```c
// Rounding operations (efficient when n is a power of 2)
// Round down to the nearest multiple of n
#define ROUNDDOWN(a, n)						\
({								\
	uint32_t __a = (uint32_t) (a);				\
	(typeof(a)) (__a - __a % (n));				\
})
// Round up to the nearest multiple of n
#define ROUNDUP(a, n)						\
({								\
	uint32_t __n = (uint32_t) (n);				\
	(typeof(a)) (ROUNDDOWN((uint32_t) (a) + __n - 1, __n));	\
})
```

#### Part2：理解文件`kern/pmap.c`中各个函数的作用，并补充完整
+ `static void *boot_alloc(uint32_t n)`
	+ 这个简单的物理内存分配器只有在JOS在设置它的虚拟内存系统时才被调用，即只会在初始化时、在`page_free_list`被设置前调用
	+ `page_alloc()`才是真正的分配函数
+ `void mem_init(void)`
	+ 设置两层页表，这个函数只设置内核部分的地址空间(>= UTOP)，用户部分的地址空间将在之后设置
	+ `UTOP`到`ULIM`：用户只能读不能写；`ULIM`之上用户不能读或写
+ `void page_init(void)`
	+ 跟踪物理页面
	+ 初始化页面结构和内存空闲链表
	+ 这个函数完成后，`boot_alloc`不能再被调用
+ `struct PageInfo *page_alloc(int alloc_flags)`
	+ 分配一个物理页面
+ `void page_free(struct PageInfo *pp)`
	+ 释放一个页面到空闲链表
+ `void page_decref(struct PageInfo* pp)`
	+ 减少对页面的引用，如果没有别的引用则释放页面
+ `pte_t *pgdir_walk(pde_t *pgdir, const void *va, int create)`
	+ 返回二级页表地址
+ `static void boot_map_region(pde_t *pgdir, uintptr_t va, size_t size, physaddr_t pa, int perm)`
	+ 映射[va, va+size]的虚拟地址空间到物理地址空间[pa, pa+size]
+ `int page_insert(pde_t *pgdir, struct PageInfo *pp, void *va, int perm)`
	+ 映射物理页面`pp`到虚拟地址`va`
+ `struct PageInfo *page_lookup(pde_t *pgdir, void *va, pte_t **pte_store)`
	+ 返回映射到虚拟内存`va`的页面
+ `void page_remove(pde_t *pgdir, void *va)`
	+ 解除在虚拟地址`va`页面的映射
