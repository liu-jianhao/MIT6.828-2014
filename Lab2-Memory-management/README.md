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
mem_init()（仅限于通话check_page_free_list(1)）
page_init()
page_alloc()
page_free()
```
`check_page_free_list()`和`check_page_alloc()`是用于测试的物理页面分配器。

启动`JOS`并查看`check_page_alloc()`报告是否成功。修复代码，使其通过。添加自己的`assert()`有助于验证假设是否正确。
