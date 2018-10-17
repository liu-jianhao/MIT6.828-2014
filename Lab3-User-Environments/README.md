# 实验3：用户环境
在本实验中将完成
+ 实现运行受保护的用户模式环境（即“进程”）所需的基本内核工具。
+ 增强JOS内核以设置数据结构以跟踪用户环境，创建单个用户环境，将程序映像加载到其中，并开始运行。
+ 使JOS内核能够处理用户环境所做的任何系统调用，并处理它导致的任何其他异常。


## 练习1：分配环境数组
`mem_init()`在`kern/pmap.c`中修改以分配和映射`envs`数组。这个数组完全由分配结构的`NENV`实例组成，`Env`就像你分配`pages`数组一样。与`pages`数组一样，内存支持`envs`也应该以只读方式映射到`UENVS（在inc/memlayout.h`中定义），以便用户进程可以从该数组中读取。
```c
// Make 'envs' point to an array of size 'NENV' of 'struct Env'.
// LAB 3: Your code here.
envs = (struct Env *) boot_alloc(NENV * sizeof(struct Env));
memset(envs, 0, NENV * sizeof(struct Env));
```

```c
// Map the 'envs' array read-only by the user at linear address UENVS
// (ie. perm = PTE_U | PTE_P).
// Permissions:
//    - the new image at UENVS  -- kernel R, user R
//    - envs itself -- kernel RW, user NONE
// LAB 3: Your code here.
boot_map_region(kern_pgdir, (uintptr_t) UENVS, ROUNDUP(NENV*sizeof(struct Env), PGSIZE), PADDR(envs), PTE_U | PTE_P);
```


## 练习2：创建和运行环境
在文件env.c中，完成以下函数的编码：

env_init()
初始化数组Env中的所有结构envs并将它们添加到env_free_list。还调用env_init_percpu，它为特权级别0（内核）和特权级别3（用户）的单独段配置分段硬件。
```c
void
env_init(void)
{
	// Set up envs array
	int i;
	for (i = NENV; i >= 0; i--) {
		envs[i].env_id = 0;
		envs[i].env_status = ENV_FREE;
		envs[i].env_link = env_free_list;
		env_free_list = &envs[i];
	}

	// Per-CPU part of the initialization
	env_init_percpu();
}
```

env_setup_vm()
为新环境分配页面目录并初始化新环境的地址空间的内核部分。
```x
static int
env_setup_vm(struct Env *e)
{
	int i;
	struct PageInfo *p = NULL;

	// Allocate a page for the page directory
	if (!(p = page_alloc(ALLOC_ZERO)))
		return -E_NO_MEM;

	// Now, set e->env_pgdir and initialize the page directory.
	//
	// Hint:
	//    - The VA space of all envs is identical above UTOP
	//	(except at UVPT, which we've set below).
	//	See inc/memlayout.h for permissions and layout.
	//	Can you use kern_pgdir as a template?  Hint: Yes.
	//	(Make sure you got the permissions right in Lab 2.)
	//    - The initial VA below UTOP is empty.
	//    - You do not need to make any more calls to page_alloc.
	//    - Note: In general, pp_ref is not maintained for
	//	physical pages mapped only above UTOP, but env_pgdir
	//	is an exception -- you need to increment env_pgdir's
	//	pp_ref for env_free to work correctly.
	//    - The functions in kern/pmap.h are handy.
	e->env_pgdir = (pde_t *) page2kva(p);
	p->pp_ref++;
	
	for (i = PDX(UTOP); i < NPDENTRIES; i++)
		e->env_pgdir[i] = kern_pgdir[i];

	// UVPT maps the env's own page table read-only.
	// Permissions: kernel R, user R
	e->env_pgdir[PDX(UVPT)] = PADDR(e->env_pgdir) | PTE_P | PTE_U;

	return 0;
}
```

region_alloc()
为环境分配和映射物理内存。要用 lab2 中的`page_insert()`完成虚拟地址到物理页的映射。
```c
static void
region_alloc(struct Env *e, void *va, size_t len)
{
	// Implement the func if you need it for load_icode
	//
	// Hint: It is easier to use region_alloc if the caller can pass
	//   'va' and 'len' values that are not page-aligned.
	//   You should round va down, and round (va + len) up.
	//   (Watch out for corner-cases!)
	char *sa = (char *) ROUNDDOWN(va, PGSIZE);
	char *ea = (char *) ROUNDUP((char *) va + len, PGSIZE);
	
	if (ea > (char *) UTOP)
		panic("region_alloc: attempting to alloc phys mem for vaddr above UTOP");
	
	for (; sa < ea; sa += PGSIZE) {
		pte_t *pte;
		struct PageInfo *pp = page_lookup(e->env_pgdir, sa, &pte);
		if (pp != NULL)  
			continue;
		
		pp = page_alloc(0);
		if (pp == NULL)  
			panic("region_alloc: out of memory for PG");
		
		if (page_insert(e->env_pgdir, pp, sa, PTE_W | PTE_U | PTE_P) < 0)
			panic("region_alloc: out of memory for PT");
	}
}
```

load_icode()
您将需要解析ELF二进制映像，就像引导加载程序已经完成的那样，并将其内容加载到新环境的用户地址空间中。
```c
static void
load_icode(struct Env *e, uint8_t *binary)
{
	// Hints:
	//  Load each program segment into virtual memory
	//  at the address specified in the ELF section header.
	//  You should only load segments with ph->p_type == ELF_PROG_LOAD.
	//  Each segment's virtual address can be found in ph->p_va
	//  and its size in memory can be found in ph->p_memsz.
	//  The ph->p_filesz bytes from the ELF binary, starting at
	//  'binary + ph->p_offset', should be copied to virtual address
	//  ph->p_va.  Any remaining memory bytes should be cleared to zero.
	//  (The ELF header should have ph->p_filesz <= ph->p_memsz.)
	//  Use functions from the previous lab to allocate and map pages.
	//
	//  All page protection bits should be user read/write for now.
	//  ELF segments are not necessarily page-aligned, but you can
	//  assume for this function that no two segments will touch
	//  the same virtual page.
	//
	//  You may find a function like region_alloc useful.
	//
	//  Loading the segments is much simpler if you can move data
	//  directly into the virtual addresses stored in the ELF binary.
	//  So which page directory should be in force during
	//  this function?
	//
	//  You must also do something with the program's entry point,
	//  to make sure that the environment starts executing there.
	//  What?  (See env_run() and env_pop_tf() below.)
	
	struct Elf *elf_hdr;
	struct Proghdr *ph, *eph;
	int i;

	lcr3(PADDR(e->env_pgdir));

	elf_hdr = (struct Elf *) binary;
	
	assert(elf_hdr->e_magic == ELF_MAGIC);
	
	ph = (struct Proghdr *) ((char *) elf_hdr + elf_hdr->e_phoff); 
	eph = ph + elf_hdr->e_phnum;

	for (; ph < eph; ph++) {
		if (ph->p_type != ELF_PROG_LOAD)
			continue;
		
		assert(ph->p_filesz <= ph->p_memsz);
		
		region_alloc(e, (void *) ph->p_va, ph->p_memsz);

		// current page directory should be e->env_pgdir
		memcpy((void *) ph->p_va, (char *) binary + ph->p_offset, ph->p_filesz);
		memset((void *) (ph->p_va + ph->p_filesz), 0, ph->p_memsz - ph->p_filesz);
	} 
	
	// 设置程序的入口地址
	e->env_tf.tf_eip = elf_hdr->e_entry;

	// Now map one page for the program's initial stack
	// at virtual address USTACKTOP - PGSIZE.
	region_alloc(e, (void *)(USTACKTOP - PGSIZE), PGSIZE);
}
```

env_create()
分配环境env_alloc 并调用load_icode加载ELF二进制文件。
```c
void
env_create(uint8_t *binary, enum EnvType type)
{
	struct Env *e;
	if (env_alloc(&e, 0) < 0 || e == NULL) 
		panic("env_create: fatal error when allocating a new env");
	e->env_type = type;
	load_icode(e, binary);
	
	// If this is the file server (type == ENV_TYPE_FS) 
	// give it I/O privileges.
	if (type == ENV_TYPE_FS)
		e->env_tf.tf_eflags |= FL_IOPL_3;
}
```

env_run()
启动以用户模式运行的给定环境。
```c
void
env_run(struct Env *e)
{
	// Step 1: If this is a context switch (a new environment is running):
	//	   1. Set the current environment (if any) back to
	//	      ENV_RUNNABLE if it is ENV_RUNNING (think about
	//	      what other states it can be in),
	//	   2. Set 'curenv' to the new environment,
	//	   3. Set its status to ENV_RUNNING,
	//	   4. Update its 'env_runs' counter,
	//	   5. Use lcr3() to switch to its address space.
	// Step 2: Use env_pop_tf() to restore the environment's
	//	   registers and drop into user mode in the
	//	   environment.

	// Hint: This function loads the new environment's state from
	//	e->env_tf.  Go back through the code you wrote above
	//	and make sure you have set the relevant parts of
	//	e->env_tf to sensible values.

	if (curenv != NULL && curenv->env_status == ENV_RUNNING)
		curenv->env_status = ENV_RUNNABLE;
	curenv = e;
	curenv->env_status = ENV_RUNNING;
	curenv->env_runs++;
  
  //  lcr3([页目录物理地址]) 将地址加载到 cr3 寄存器。
	lcr3(PADDR(curenv->env_pgdir));

	unlock_kernel();
	env_pop_tf(&curenv->env_tf);
}
```

# 处理中断和异常

## 练习4 编辑trapentry.S和trap.c并实现下述功能
每个异常或中断都应该在trapentry.S中有自己的处理程序， 并且trap_init()应该使用这些处理程序的地址初始化IDT。
每个处理程序都应该在堆栈上构建一个struct Trapframe（参见inc/trap.h）并使用指向Trapframe的指针调用 trap()（在trap.c中）。
trap()然后处理异常/中断或调度到特定的处理函数。

在`trapentry.S`中：
```c
TRAPHANDLER_NOEC(handler0, T_DIVIDE)
TRAPHANDLER_NOEC(handler1, T_DEBUG)
TRAPHANDLER_NOEC(handler2, T_NMI)
TRAPHANDLER_NOEC(handler3, T_BRKPT)
TRAPHANDLER_NOEC(handler4, T_OFLOW)
TRAPHANDLER_NOEC(handler5, T_BOUND)
TRAPHANDLER_NOEC(handler6, T_ILLOP)
TRAPHANDLER_NOEC(handler7, T_DEVICE)
TRAPHANDLER(handler8, T_DBLFLT)
// 9 deprecated since 386
TRAPHANDLER(handler10, T_TSS)
TRAPHANDLER(handler11, T_SEGNP)
TRAPHANDLER(handler12, T_STACK)
TRAPHANDLER(handler13, T_GPFLT)
TRAPHANDLER(handler14, T_PGFLT)
// 15 reserved by intel
TRAPHANDLER_NOEC(handler16, T_FPERR)
TRAPHANDLER(handler17, T_ALIGN)
TRAPHANDLER_NOEC(handler18, T_MCHK)
TRAPHANDLER_NOEC(handler19, T_SIMDERR)
// system call (interrupt)
TRAPHANDLER_NOEC(handler48, T_SYSCALL)
```
该部分主要作用是声明函数。该函数是全局的，但是在 C 文件中使用的时候需要使用 void name(); 再声明一下。
```
_alltraps:
pushl %ds
pushl %es
pushal

movw $GD_KD, %ax
movw %ax, %ds
movw %ax, %es
pushl %esp
call trap
```
栈是从高地址向低地址生长，而结构体在内存中的存储是从低地址到高地址。而 cpu 以及TRAPHANDLER宏已经将压栈工作进行到了中断向量部分，若要形成一个 Trapframe，则还应该依次压入 ds, es以及 struct PushRegs中的各寄存器（倒序，可使用 pusha指令）。此后还需要更改数据段为内核的数据段。 

**注意，不能用立即数直接给段寄存器赋值。** 因此不能直接写`movw $GD_KD, %ds`。

在kern/trap.c 中：
```c
void
trap_init(void)
{
	extern struct Segdesc gdt[];

	// LAB 3: Your code here.
	void handler0();
	void handler1();
	void handler2();
	void handler3();
	void handler4();
	void handler5();
	void handler6();
	void handler7();
	void handler8();

	void handler10();
	void handler11();
	void handler12();
	void handler13();
	void handler14();

	void handler16();
	void handler17();
	void handler18();
	void handler19();
	void handler48();

	SETGATE(idt[T_DIVIDE], 1, GD_KT, handler0, 0);
	SETGATE(idt[T_DEBUG], 1, GD_KT, handler1, 0);
	SETGATE(idt[T_NMI], 1, GD_KT, handler2, 0);
	SETGATE(idt[T_BRKPT], 1, GD_KT, handler3, 3);
	SETGATE(idt[T_OFLOW], 1, GD_KT, handler4, 0);
	SETGATE(idt[T_BOUND], 1, GD_KT, handler5, 0);
	SETGATE(idt[T_ILLOP], 1, GD_KT, handler6, 0);
	SETGATE(idt[T_DEVICE], 1, GD_KT, handler7, 0);
	SETGATE(idt[T_DBLFLT], 1, GD_KT, handler8, 0);

	SETGATE(idt[T_TSS], 1, GD_KT, handler10, 0);
	SETGATE(idt[T_SEGNP], 1, GD_KT, handler11, 0);
	SETGATE(idt[T_STACK], 1, GD_KT, handler12, 0);
	SETGATE(idt[T_GPFLT], 1, GD_KT, handler13, 0);
	SETGATE(idt[T_PGFLT], 1, GD_KT, handler14, 0);
	
	SETGATE(idt[T_FPERR], 1, GD_KT, handler16, 0);
	SETGATE(idt[T_ALIGN], 1, GD_KT, handler17, 0);
	SETGATE(idt[T_MCHK], 1, GD_KT, handler18, 0);
	SETGATE(idt[T_SIMDERR], 1, GD_KT, handler19, 0);

	// interrupt
	SETGATE(idt[T_SYSCALL], 0, GD_KT, handler48, 0);
	
	// Per-CPU setup 
	trap_init_percpu();
}
```


## 练习5 修改trap_dispatch() 以将页面错误异常分派给page_fault_handler()
## 练习6. 修改trap_dispatch() 以使断点异常调用内核监视器

```c
static void
trap_dispatch(struct Trapframe *tf)
{
    // Handle processor exceptions.
    // LAB 3: Your code here.
    switch (tf->tf_trapno) {
        case T_PGFLT:
            page_fault_handler(tf);
            break;
        case T_BRKPT:
            monitor(tf);
            break;
        default:
          // Unexpected trap: The user process or the kernel has a bug.
          print_trapframe(tf);
          if (tf->tf_cs == GD_KT)
              panic("unhandled trap in kernel");
          else {
              env_destroy(curenv);
              return;
          }
    }
}
```

# 系统调用

## 练习7. 在内核中为中断向量添加一个处理程序T_SYSCALL
在`trap_dispatch`函数中加入相应的处理方法：
```c
case T_SYSCALL:
  tf->tf_regs.reg_eax = syscall(tf->tf_regs.reg_eax, 
          tf->tf_regs.reg_edx,
          tf->tf_regs.reg_ecx,
          tf->tf_regs.reg_ebx,
          tf->tf_regs.reg_edi,
          tf->tf_regs.reg_esi);
  break;
```

在`kern/trap.c`中
```c
int32_t
syscall(uint32_t syscallno, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5)
{
	// Call the function corresponding to the 'syscallno' parameter.
	// Return any appropriate return value.
	// LAB 3: Your code here.

	// panic("syscall not implemented");
	
	int32_t retVal = 0;
	switch (syscallno) {
	case SYS_cputs:
		sys_cputs((const char *)a1, a2);
		break;
	case SYS_cgetc:
		retVal = sys_cgetc();
		break;
	case SYS_env_destroy:
		retVal = sys_env_destroy(a1);
		break;
	case SYS_getenvid:
		retVal = sys_getenvid() >= 0;
		break;
	default:
		retVal = -E_INVAL;
	}
	return retVal;
}
```

## 练习8. 将所需代码添加到用户库，然后启动内核
## 练习9. kern/trap.c如果在内核模式下发生页面错误，更改为panic。
## 练习10. 启动内核，运行user/evilhello。应该破坏环境，内核不应该panic。
在 kern/pmap.c 中修改检查用户内存的部分。需要注意的是由于需要存储第一个访问出错的地址，va 所在的页面需要单独处理一下，不能直接对齐。
```c
int
user_mem_check(struct Env *env, const void *va, size_t len, int perm)
{
	// LAB 3: Your code here.
	uintptr_t start_va = ROUNDDOWN((uintptr_t)va, PGSIZE);
	uintptr_t end_va = ROUNDUP((uintptr_t)va + len, PGSIZE);
	for (uintptr_t cur_va=start_va; cur_va<end_va; cur_va+=PGSIZE) {
		pte_t *cur_pte = pgdir_walk(env->env_pgdir, (void *)cur_va, 0);
		if (cur_pte == NULL || (*cur_pte & (perm|PTE_P)) != (perm|PTE_P) || cur_va >= ULIM) {
			if (cur_va == start_va) {
				user_mem_check_addr = (uintptr_t)va;
			} else {
				user_mem_check_addr = cur_va;
			}
			return -E_FAULT;
		}
	}
	return 0;
}
```


在 kern/syscall.c 中的输出字符串部分加入内存检查。
```c
static void
sys_cputs(const char *s, size_t len)
{
	// Check that the user has permission to read memory [s, s+len).
	// Destroy the environment if not.

	// LAB 3: Your code here.
	user_mem_assert(curenv, s, len, PTE_U);
	// Print the string supplied by the user.
	cprintf("%.*s", len, s);
}
```

在 kern/kdebug.c 中的 debuginfo_eip 函数中加入内存检查。
```c
		// Make sure this memory is valid.
		// Return -1 if it is not.  Hint: Call user_mem_check.
		// LAB 3: Your code here.
		if (user_mem_check(curenv, (void *)usd, sizeof(struct UserStabData), PTE_U) < 0) {
			return -1;
		}
...
		// Make sure the STABS and string table memory is valid.
		// LAB 3: Your code here.
		if (user_mem_check(curenv, (void *)stabs, stab_end-stabs, PTE_U) < 0) {
			return -1;
		}
		if (user_mem_check(curenv, (void *)stabstr, stabstr_end-stabstr, PTE_U) < 0) {
			return -1;
		}
```

## 总结
这样就完成了实验三了。
