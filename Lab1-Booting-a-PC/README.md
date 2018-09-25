[实验一指导](https://pdos.csail.mit.edu/6.828/2014/labs/lab1/#Hand-In-Procedure)
## 环境搭建
环境搭建是最重要的一步，不然后面的实验会有很多莫名其妙的错误

[环境搭建](https://zhuanlan.zhihu.com/p/36889298)

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
#### 在`BIOS`进入引导加载程序时检查内存的8个字在`0x00100000`处，然后在引导加载程序进入内核时再次检查。他们为什么不同？第二个断点有什么？

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
#### 使用QEMU和GDB跟踪到JOS内核并停在`movl %eax, %cr0`。检查内存为`0x00100000`和`0xf0100000`。现在，使用`stepi`GDB命令单步执行该指令。再次，检查内存为`0x00100000`和`0xf0100000`。确保你了解刚刚发生的事情。

先将断点设置到加载内核的前一句
```
(gdb) b *0x7d6b
Breakpoint 1 at 0x7d6b
(gdb) c
Continuing.
The target architecture is assumed to be i386
=> 0x7d6b:	call   *0x10018
```
然后进入内核并停在`movl %eax, %cr0`，查看地址`0x100000`和`0xf0100000`的内容，可以发现内容不一样
```
(gdb) 
=> 0x100025:	mov    %eax,%cr0
0x00100025 in ?? ()
(gdb) x/8x 0x100000
0x100000:	0x1badb002	0x00000000	0xe4524ffe	0x7205c766
0x100010:	0x34000004	0x2000b812	0x220f0011	0xc0200fd8
(gdb) x/8x 0xf0100000
0xf0100000 <_start+4026531828>:	0x00000000	0x00000000	0x00000000	0x00000000
0xf0100010 <entry+4>:	0x00000000	0x00000000	0x00000000	0x00000000
```
继续执行，执行完`movl %eax, %cr0`后，可以发现，`VMA`与`LMA`现在具有同样的内容。这是因为`0x00100000`被映射到了`0xf0100000`处
```
(gdb) si
=> 0x100028:	mov    $0xf010002f,%eax
0x00100028 in ?? ()
(gdb) x/8x 0x100000
0x100000:	0x1badb002	0x00000000	0xe4524ffe	0x7205c766
0x100010:	0x34000004	0x2000b812	0x220f0011	0xc0200fd8
(gdb) x/8x 0xf0100000
0xf0100000 <_start+4026531828>:	0x1badb002	0x00000000	0xe4524ffe	0x7205c766
0xf0100010 <entry+4>:	0x34000004	0x2000b812	0x220f0011	0xc0200fd8
```

#### 建立新映射后，如果映射不到位，将无法正常工作的第一条指令是什么？`movl %eax, %cr0`在 `kern/entry.S`中注释掉，跟踪它，看看你是否正确。
程序会直接崩溃，这是在执行完`jmp    *%eax`后崩溃的
```
(gdb) 
=> 0x10002a:	jmp    *%eax
0x0010002a in ?? ()
(gdb)
=> 0xf010002c <relocated>:	add    %al,(%eax)
relocated () at kern/entry.S:74
74		movl	$0x0,%ebp			# nuke frame pointer
(gdb) 
Remote connection closed
```

### 练习8
#### 我们省略了一小段代码——使用“％o”形式的模式打印八进制数所需的代码。查找并填写此代码片段。
要补充的代码在`lib/printfmt.c`
```c
// (unsigned) octal
case 'o':
	// Replace this with your code.
	putch('X', putdat);
	putch('X', putdat);
	putch('X', putdat);
	break;
```
参考上面十进制的输出，替换为
```c
// (unsigned) octal
case 'o':
	// Replace this with your code.
	num = getuint(&ap, lflag);
	base = 8;
	goto number;
```

#### 解释`printf.c`和`console.c`之间的接口。具体来说，`console.c`导出什么功能？`printf.c`如何使用此函数？


#### 从`console.c`解释以下内容
内容在`console.c`的`cga_putc`函数里
```c
if (crt_pos >= CRT_SIZE) {
	int i;
	// 把从第1~n行的内容复制到0~(n-1)行，第n行未变化
	memmove(crt_buf, crt_buf + CRT_COLS, (CRT_SIZE - CRT_COLS) * sizeof(uint16_t));
	// 将第n行覆盖为默认属性下的空格
	for (i = CRT_SIZE - CRT_COLS; i < CRT_SIZE; i++)
		crt_buf[i] = 0x0700 | ' ';
	// 清空了最后一行，同步crt_pos
	crt_pos -= CRT_COLS;
}
```
看完这个函数大概可以猜出来这个函数应该是在屏幕上输出字符，而显示屏幕是有大小的，`crt_pos`是当前光标的位置，而`CRT_SIZE`是显示屏幕的大小，超过这个大小，则要向下移动一行

#### 逐步跟踪以下代码的执行：
```c
int x = 1，y = 3，z = 4;
cprintf（“x％d，y％x，z％d \ n”，x，y，z）;
```
+ 在调用`cprintf()`时，到底`fmt`有什么意义？到什么`ap`点？
`fmt`是`cprintf`函数的第一个参数，即指向字符串`"x %d, y %x, z %d\n"`的指针

`ap`指向第二个参数的地址。**注意`ap`中存放的是第二个参数的地址，而非第二个参数。**
+ 列出（按执行顺序）每次调用 `cons_putc`，`va_arg`和`vcprintf`。对于`cons_putc`，也列出其论点。对于`va_arg`，列出`ap`呼叫之前和之后的点。对于`vcprintf`名单的两个参数的值。
调用关系为`cprintf -> vcprintf -> vprintfmt -> putch -> cputchar -> cons_putc`


#### 运行以下代码
```c
unsigned int i = 0x00646c72;
cprintf（“H％x Wo％s”，57616，＆i）;
```
+ 什么是输出？解释如何以前一个练习的逐步方式获得此输出。
输出是He110 World。 57616的16进制形式为 e110，这个很好理解。 输出字符串时，从给定字符串的第一个字符地址开始，按字节读取字符，直到遇到 '\0' 结束。

于是，Wo%s, &i 的意义是把 i 作为字符串输出。查阅 ASCII 码表可知，0x00 对应 '\0'，0x64 对应 'd'，0x6c 对应 'l'，0x72 对应 'r'。


#### 在下面的代码中，将要打印的是什么 'y='？（注意：答案不是特定值。）为什么会发生这种情况？
```c
cprintf（“x =％dy =％d”，3）;
```
输出为：`x = 3, y = -267321588`。 由于第二个参数尚未指定，输出3以后无法确定`ap`的值应该变化多少，更无法根据`ap`的值获取参数。`va_arg`取当前栈地址，并将指针移动到下个“参数”所在位置简单的栈内移动，没有任何标志或者条件能够让你确定可变参函数的参数个数，也不能判断当前栈指针的合法性。

#### 假设GCC更改了它的调用约定，以便它按声明顺序在堆栈上推送参数，以便最后推送最后一个参数。您将如何更改cprintf或其界面，以便仍然可以传递可变数量的参数？
需要更改`va_start`以及`va_arg`两个宏的实现。

### 练习9
#### 确定内核初始化其堆栈的位置，以及堆栈所在内存的确切位置。内核如何为其堆栈保留空间？并且在这个保留区域的“结束”是堆栈指针初始化为指向？
在`kern/entry.S`中找到初始化`ebp`和`esp`的语句：
```
# Clear the frame pointer register (EBP)
# so that once we get into debugging C code,
# stack backtraces will be terminated properly.
movl	$0x0,%ebp			# nuke frame pointer

# Set the stack pointer
movl	$(bootstacktop),%esp
```
用gdb查看地址：
```
(gdb) b kern/entry.S:74
Breakpoint 1 at 0xf010002f: file kern/entry.S, line 74.
(gdb) c
Continuing.
The target architecture is assumed to be i386
=> 0xf010002f <relocated>:	mov    $0x0,%ebp

Breakpoint 1, relocated () at kern/entry.S:74
74		movl	$0x0,%ebp			# nuke frame pointer
(gdb) si
=> 0xf0100034 <relocated+5>:	mov    $0xf0110000,%esp
relocated () at kern/entry.S:77
77		movl	$(bootstacktop),%esp
```
可以看出，栈顶在`0xf0110000`，然后在`kern/entry.S`中找到：
```
bootstack:
	.space		KSTKSIZE
```
在`inc/memlayout.h`中找到以下定义：
```
// Kernel stack.
#define KSTACKTOP	KERNBASE
#define KSTKSIZE	(8*PGSIZE)   		// size of a kernel stack
#define KSTKGAP		(8*PGSIZE)   		// size of a kernel stack guard
```
在`inc/mmu.h`中找到以下定义：
```
#define PGSIZE		4096		// bytes mapped by a page
```
可以看出，栈大小为32kB。

由于栈是从内存高位向低位生长，所以堆栈指针指向的是高位。

### 练习10
#### 要熟悉x86上的C调用约定，`test_backtrace`在`obj/kern/kernel.asm`中找到函数的地址，在那里设置断点，并检查每次在内核启动后调用它时会发生什么。每个递归嵌套级别`test_backtrace`的堆栈中有多少32位字，这些字是什么？

`test_backtrace`函数在`kern/init.c`里定义和使用
```c
// Test the stack backtrace function (lab 1 only)
void
test_backtrace(int x)
{
	cprintf("entering test_backtrace %d\n", x);
	if (x > 0)
		test_backtrace(x-1);
	else
		mon_backtrace(0, 0, 0);
	cprintf("leaving test_backtrace %d\n", x);
}
```
```c
// Test the stack backtrace function (lab 1 only)
test_backtrace(5);
```
开始调试：
```
(gdb) b *0xf0100076
Breakpoint 1 at 0xf0100076: file kern/init.c, line 18.
(gdb) c
Continuing.
The target architecture is assumed to be i386
=> 0xf0100076 <test_backtrace+54>:	call   0xf010076e <mon_backtrace>
Breakpoint 1, 0xf0100076 in test_backtrace (x=0) at kern/init.c:18
18			mon_backtrace(0, 0, 0);
(gdb) x/52x $esp
0xf010ff20:	0x00000000	0x00000000	0x00000000	0x00000000
0xf010ff30:	0xf01008ef	0x00000001	0xf010ff58	0xf0100068
0xf010ff40:	0x00000000	0x00000001	0xf010ff78	0x00000000
0xf010ff50:	0xf01008ef	0x00000002	0xf010ff78	0xf0100068
0xf010ff60:	0x00000001	0x00000002	0xf010ff98	0x00000000
0xf010ff70:	0xf01008ef	0x00000003	0xf010ff98	0xf0100068
0xf010ff80:	0x00000002	0x00000003	0xf010ffb8	0x00000000
0xf010ff90:	0xf01008ef	0x00000004	0xf010ffb8	0xf0100068
0xf010ffa0:	0x00000003	0x00000004	0x00000000	0x00000000
0xf010ffb0:	0x00000000	0x00000005	0xf010ffd8	0xf0100068
0xf010ffc0:	0x00000004	0x00000005	0x00000000	0x00010094
0xf010ffd0:	0x00010094	0x00010094	0xf010fff8	0xf01000d4
0xf010ffe0:	0x00000005	0x00001aac	0x00000644	0x00000000
0xf010fff0:	0x00000000	0x00000000	0x00000000	0xf010003e
```
因为栈向下生长，从后往前看即为执行顺序。 在调用函数时，对栈需要进行以下操作：
1. 将参数由右向左压入栈
2. 将返回地址 (eip中的内容) 入栈，在 call 指令执行
3. 将上一个函数的 ebp 入栈
4. 将 ebx 入栈，保护寄存器状态
5. 在栈上开辟一个空间存储局部变量
可以看出，第二列出现的`0x00000005`到`0x00000000`都是参数。 在参数前一个存储的是返回地址，`0xf0100068`出现了多次，是`test_backtrace`递归过程中的返回地址。而`0xf01000d4`出现仅一次，是`i386_init`函数中的返回地址。可以通过查看`obj/kern/kernel.asm`证明。

### 练习11
#### 实现上面指定的回溯函数。使用与示例中相同的格式，否则将使评分脚本混淆。如果您认为它正常工作，请运行make grade以查看其输出是否符合我们的评分脚本所期望的内容，如果不符合则修复它。 在您交付Lab 1代码后，欢迎您以任何方式更改回溯功能的输出格式。
输出格式为：
```
Stack backtrace:
  ebp f0109e58  eip f0100a62  args 00000001 f0109e80 f0109e98 f0100ed2 00000031
  ebp f0109ed8  eip f01000d6  args 00000000 00000000 f0100058 f0109f28 00000061
  ...
```

主要是根据提示来改写`kern/monitor.c`，要点：
1. 利用read_ebp() 函数获取当前ebp值
2. 利用 ebp 的初始值0判断是否停止
3. 利用数组指针运算来获取 eip 以及 args
```c
int
mon_backtrace(int argc, char **argv, struct Trapframe *tf)
{
	// Your code here.
	uint32_t ebp, *p;
	ebp = read_ebp();
	while(ebp != 0)
	{
		p = (uint32_t*)ebp;
		cprintf("ebp %x eip %x args %08x %08x %08x %08x %08x\n", ebp, p[1], p[2], p[3], p[4], p[5], p[6]);
		ebp = *p;
	}
	return 0;
}
```

### 练习12
#### 修改堆栈回溯功能，为每个eip显示与该eip对应的函数名称，源文件名和行号。
输出格式为：
```
K> backtrace
Stack backtrace:
  ebp f010ff78  eip f01008ae  args 00000001 f010ff8c 00000000 f0110580 00000000
         kern/monitor.c:143: monitor+106
  ebp f010ffd8  eip f0100193  args 00000000 00001aac 00000660 00000000 00000000
         kern/init.c:49: i386_init+59
  ebp f010fff8  eip f010003d  args 00000000 00000000 0000ffff 10cf9a00 0000ffff
         kern/entry.S:70: <unknown>+0
K>
```


首先是完成二分查找`stab`表确定行号的函数，在`kern/kdebug.c`的173行处 根据注释的提示基本就能完成：
```c
// Search within [lline, rline] for the line number stab.
// If found, set info->eip_line to the right line number.
// If not found, return -1.
//
// Hint:
//	There's a particular stabs type used for line numbers.
//	Look at the STABS documentation and <inc/stab.h> to find
//	which one.
// Your code here.
stab_binsearch(stabs, &lline, &rline, N_SLINE, addr);
if(lline <= rline)
{
	info->eip_line = stabs[lline].n_desc;
}
else
{
	return -1;
}
```
此后是添加命令，在 kern/monitor.c 的第27行：
```c
static struct Command commands[] = {
	{ "help", "Display this list of commands", mon_help },
	{ "kerninfo", "Display information about the kernel", mon_kerninfo },
	{ "backtrace", "Display information about the backtrace", mon_backtrace },
};
```
最后是添加`backtrace`的输出信息，将`kern/monitor.c`的`mon_backtrace`函数改为：
```c
int
mon_backtrace(int argc, char **argv, struct Trapframe *tf)
{
	// Your code here.
	uint32_t ebp, *p;
	struct Eipdebuginfo eip_info;
	ebp = read_ebp();
	while(ebp != 0)
	{
		p = (uint32_t*)ebp;
		cprintf("ebp %x eip %x args %08x %08x %08x %08x %08x\n", ebp, p[1], p[2], p[3], p[4], p[5], p[6]);
		if(debuginfo_eip(p[1], &eip_info) == 0)
		{
			uint32_t offset = p[1] - eip_info.eip_fn_addr;
			cprintf("\t\t%s:%d: %.*s+%d\n", eip_info.eip_file, eip_info.eip_line, eip_info.eip_fn_namelen,  eip_info.eip_fn_name, offset);
		}
		ebp = *p;
	}
	return 0;
}
```

### 检测
```shell
$ make grade
make clean
make[1]: Entering directory '/home/liu/Desktop/6.828/lab'
rm -rf obj .gdbinit jos.in qemu.log
make[1]: Leaving directory '/home/liu/Desktop/6.828/lab'
./grade-lab1
make[1]: Entering directory '/home/liu/Desktop/6.828/lab'
+ as kern/entry.S
+ cc kern/entrypgdir.c
+ cc kern/init.c
+ cc kern/console.c
+ cc kern/monitor.c
+ cc kern/printf.c
+ cc kern/kdebug.c
+ cc lib/printfmt.c
+ cc lib/readline.c
+ cc lib/string.c
+ ld obj/kern/kernel
+ as boot/boot.S
+ cc -Os boot/main.c
+ ld boot/boot
boot block is 380 bytes (max 510)
+ mk obj/kern/kernel.img
make[1]: Leaving directory '/home/liu/Desktop/6.828/lab'
running JOS: (1.1s)
  printf: OK
  backtrace count: OK
  backtrace arguments: OK
  backtrace symbols: OK
  backtrace lines: OK
Score: 50/50
```
