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
$ objdump -p obj/kern/kernel                                                                                                           lab1 [882640d]

obj/kern/kernel:     file format elf32-i386

Program Header:
    LOAD off    0x00001000 vaddr 0xf0100000 paddr 0x00100000 align 2**12
         filesz 0x0000759d memsz 0x0000759d flags r-x
    LOAD off    0x00009000 vaddr 0xf0108000 paddr 0x00108000 align 2**12
         filesz 0x0000b044 memsz 0x0000b6a4 flags rw-
   STACK off    0x00000000 vaddr 0x00000000 paddr 0x00000000 align 2**4
         filesz 0x00000000 memsz 0x00000000 flags rwx
```
