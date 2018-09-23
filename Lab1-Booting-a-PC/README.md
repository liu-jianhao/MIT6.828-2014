[实验一指导](https://pdos.csail.mit.edu/6.828/2014/labs/lab1/#Hand-In-Procedure)

## 实验步骤
1. 先按照指导，下载源码并编译链接
2. 开始做实验

## 第1部分：PC Bootstrap
看指导书

## 第2部分：引导加载程序
### 笔记
+ PC的软盘和硬盘分为512个字节区域，称为扇区。扇区是磁盘的最小传输粒度：每个读取或写入操作必须是一个或多个扇区，并在扇区边界上对齐。
+ 如果磁盘是可引导的，则第一个扇区称为引导扇区，因为这是引导加载程序代码所在的位置。
+ 当`BIOS`找到可引导的软盘或硬盘时，它会将512字节的引导扇区加载到物理地址`0x7c00`到`0x7dff`的内存中，
然后使用`jmp`指令将`CS:IP`设置为`0000:7c00`，将控制权交给引导装载机。与`BIOS`加载地址一样，这些地址相当随意——但它们是针对PC修复和标准化的。

### 实验准备
对于本实验，引导加载程序由一个汇编语言源文件`boot/boot.S`和一个C源文件`boot/main.c`组成，我们先阅读这两个文件的源码

然后学习了解这次要用到的[工具](https://pdos.csail.mit.edu/6.828/2014/labguide.html)

### 练习3
#### 处理器在什么时候开始执行32位代码？究竟是什么导致从16位模式切换到32位模式？
在`boot/boot.S`中，我们可以看到这样一行代码
```
  # Jump to next instruction, but in 32-bit code segment.
  # Switches processor into 32-bit mode.
  ljmp    $PROT_MODE_CSEG, $protcseg
```
在这一步，执行了一个段间跳转指令，格式为`ljmp $SECTION, $OFFSET`，并且从此开始执行32位代码。


用`si`命令一步一步到有`ljmp`这一步

以下指令导致了从实模式到保护模式到转换。`cr0`寄存器的0位置1。
```
(gdb) 
[   0:7c23] => 0x7c23:	mov    %cr0,%eax
0x00007c23 in ?? ()
(gdb) 
[   0:7c26] => 0x7c26:	or     $0x1,%ax
0x00007c26 in ?? ()
(gdb) 
[   0:7c2a] => 0x7c2a:	mov    %eax,%cr0
0x00007c2a in ?? ()
(gdb) 
[   0:7c2d] => 0x7c2d:	ljmp   $0xb866,$0x87c32
0x00007c2d in ?? ()
```


#### 引导加载程序执行的最后一条指令是什么，它刚加载的内核的第一条指令是什么？
在`boot/main.c`中可以到这一行代码
```c
	// call the entry point from the ELF header
	// note: does not return!
	((void (*)(void)) (ELFHDR->e_entry))();
```
这就是引导加载程序执行的最后一条指令，我们可以在gdb找出来，地址为`0x7d6b`
```
(gdb) b *0x7d6b
Breakpoint 3 at 0x7d6b
(gdb) c
Continuing.
=> 0x7d6b:	call   *0x10018
```
接下去执行的就是内核执行的第一条指令：
```
(gdb) si
=> 0x10000c:	movw   $0x1234,0x472
0x0010000c in ?? ()
```

#### 内核的第一条指令在哪里？
由上一问可以知道就是地址`0x10000c`

#### 引导加载程序如何决定从磁盘获取整个内核必须读取多少扇区？它在哪里找到这些信息？
根据对`main.c`的分析，显然是通过 ELF 文件头获取所有`program header table`

通过 objdump 命令可以查看：
```
$ objdump -p obj/kern/kernel                                                                                                   

obj/kern/kernel:     file format elf32-i386

Program Header:
    LOAD off    0x00001000 vaddr 0xf0100000 paddr 0x00100000 align 2**12
         filesz 0x0000759d memsz 0x0000759d flags r-x
    LOAD off    0x00009000 vaddr 0xf0108000 paddr 0x00108000 align 2**12
         filesz 0x0000b044 memsz 0x0000b6a4 flags rw-
   STACK off    0x00000000 vaddr 0x00000000 paddr 0x00000000 align 2**4
         filesz 0x00000000 memsz 0x00000000 flags rwx
```


### 练习5
+ 再次跟踪引导加载程序的前几条指令，并确定第一条指令执行错误操作，如果您要使引导加载程序的链接地址错误。然后将`boot/Makefrag`中的链接地址更改为错误的地址，运行`make clean`，重新编译实验`make`，然后再次跟踪到引导加载程序以查看发生的情况。

将`boot/Makefrag`中的`-Ttext 0x7c00`改为`-Ttext 0x7c20`，仍然将断点设置在`0x7c00`
```
(gdb) 
[   0:7c2d] => 0x7c2d:	ljmp   $0xb866,$0x87c52
0x00007c2d in ?? ()
```
与之前对比可以发现，相差了`0x20`，也就是修改增加的值。然而由于`BIOS`会把引导加载程序固定加载在`0x7c00`，于是导致了错误。


### 练习6
在`BIOS`进入引导加载程序时检查内存的8个字在`0x00100000`处，然后在引导加载程序进入内核时再次检查。他们为什么不同？第二个断点有什么？

ELF头中有一个很重要的字段，名为`e_entry`。该字段保存程序中入口点的链接地址：程序应该开始执行的程序文本部分中的内存地址。你可以看到入口点：
```
$ objdump -f obj/kern/kernel 

obj/kern/kernel:     file format elf32-i386
architecture: i386, flags 0x00000112:
EXEC_P, HAS_SYMS, D_PAGED
start address 0x0010000c
```
这和练习3的内核的第一条指令的位置相符。 可以看出，`boot/main.c`的作用就是从硬盘读取内核的每个段，然后跳转到内核的`e_entry`。

首先在`BIOS`进入引导加载程序时检查一次（还未读取内核至内存），再在从引导加载程序进入内核时检查一次（此时已经将内核读入内存）。

```
(gdb) b *0x7c00
Breakpoint 1 at 0x7c00
(gdb) c
Continuing.
[   0:7c00] => 0x7c00:	cli    

Breakpoint 1, 0x00007c00 in ?? ()
(gdb) x/8x 0x100000
0x100000:	0x00000000	0x00000000	0x00000000	0x00000000
0x100010:	0x00000000	0x00000000	0x00000000	0x00000000
(gdb) b *0x7d6b
Breakpoint 2 at 0x7d6b
(gdb) c
Continuing.
The target architecture is assumed to be i386
=> 0x7d6b:	call   *0x10018

Breakpoint 2, 0x00007d6b in ?? ()
(gdb) x/8x 0x100000
0x100000:	0x1badb002	0x00000000	0xe4524ffe	0x7205c766
0x100010:	0x34000004	0x2000b812	0x220f0011	0xc0200fd8
```


### 练习7
