在这个lab中，我们会实现能让受保护的用户模式环境（进程）运行所需的基本内核功能，即为了能让用户进程运行在操作系统上，操作系统需要实现哪些功能

我们需要建立数据结构来追踪用户环境，创建一个单一的用户环境，加载一个程序镜像进进程，然后让它运行起来。用户环境和进程的概念可以互换，在jos中我们称之为环境，后文不再赘述。  

同时我们还会让JOS kernel更加强大，使之可以处理任何用户环境造成的系统调用和其他异常。  

### Getting Started

Lab3中包含了大量的新创建的源文件，以下都需要浏览一遍。

| `inc/`  | `env.h`       | Public definitions for user-mode environments                |
| ------- | ------------- | ------------------------------------------------------------ |
|         | `trap.h`      | Public definitions for trap handling                         |
|         | `syscall.h`   | Public definitions for system calls from user environments to the kernel |
|         | `lib.h`       | Public definitions for the user-mode support library         |
| `kern/` | `env.h`       | Kernel-private definitions for user-mode environments        |
|         | `env.c`       | Kernel code implementing user-mode environments              |
|         | `trap.h`      | Kernel-private trap handling definitions                     |
|         | `trap.c`      | Trap handling code                                           |
|         | `trapentry.S` | Assembly-language trap handler entry-points                  |
|         | `syscall.h`   | Kernel-private definitions for system call handling          |
|         | `syscall.c`   | System call implementation code                              |
| `lib/`  | `Makefrag`    | Makefile fragment to build user-mode library, `obj/lib/libjos.a` |
|         | `entry.S`     | Assembly-language entry-point for user environments          |
|         | `libmain.c`   | User-mode library setup code called from `entry.S`           |
|         | `syscall.c`   | User-mode system call stub functions                         |
|         | `console.c`   | User-mode implementations of `putchar` and `getchar`, providing console I/O |
|         | `exit.c`      | User-mode implementation of `exit`                           |
|         | `panic.c`     | User-mode implementation of `panic`                          |
| `user/` | `*`           | Various test programs to check kernel lab 3 code             |

### Inline Assembly

在这个lab中，你会发现GCC的内联汇编十分有帮助，尽管不使用内联汇编也可以完成lab。但至少，你需要可以理解内联汇编语言的代码段，如果想学习内联汇编，可以参考官方的参考资料。  

## Part A: User Environments and Exception Handling

新的头文件`inc/env.h`包含了JOS中用户环境的基本定义。内核使用数据结构`Env`来追踪每个用户环境。在本次lab中我们最初将仅创建一个用户环境，但你需要设计内核来使之支持多环境。lab4会通过一个用户环境`fork`其他环境来实现这个特性。  

在`kern/env.c`中，内核主要维护三个与环境有关的全局变量：

```c
struct Env *envs = NULL;		// All environments
struct Env *curenv = NULL;		// The current env
static struct Env *env_free_list;	// Free environment list
```

一旦JOS启动并运行，`envs`指针就会指向一个代表系统中所有环境的`Env`结构体数组。在我们的设计中，JOS内核可以支持最大`NENV`个同时运行的环境，尽管在一段特定的时间内不会有这么多运行的环境。(`NENV`是一个常数，定义在`inc/env.h`)。一旦环境被分配内存，`envs`数组就会包含一个`Env`结构体的实例。  

JOS内核会用一个数据结构`env_free_list`来记录所有不活动的`Env`结构。这个设计能让分配和撤销分配环境变得十分简单，因为仅仅只需要从链表中插入和移除一个结点就行。  

内核使用`curenv`符号来追踪在任何给定时间当前正在执行的环境。在启动过程中，在第一个环境运行之前，毫无疑问`curenv`是一个指向`NULL`的指针。 

### Environment State

`Env`结构体定义如下：

```c
struct Env {
	struct Trapframe env_tf;	// Saved registers
	struct Env *env_link;		// Next free Env
	envid_t env_id;			// Unique environment identifier
	envid_t env_parent_id;		// env_id of this env's parent
	enum EnvType env_type;		// Indicates special system environments
	unsigned env_status;		// Status of the environment
	uint32_t env_runs;		// Number of times environment has run

	// Address space
	pde_t *env_pgdir;		// Kernel virtual address of page dir
};
```

**env_tf**:这个结构定义在`inc/trap.h`中，保存了当环境没有运行时，即当内核或者一个其他的环境正在运行时saved register的值。当内核从用户模式转为内核模式时会保存这些值，以便环境之后能从它终端的地方恢复。  

**env_link**:这是指向`env_free_list`上的下一个链接，`env_free_list`指向第一个可用的环境。  

**env_id**:内核负责赋值给这个字段，它唯一标识了当前使用此`Env`结构的环境。即使用`envs`数组中一个特定项。在这个环境终止后，内核可能会重新分配相同的Env结构给不同的环境使用，也就是说地址还是同样的，不过新的环境会拥有一个完全不同的`env_id`，尽管它们使用的是`envs`数组中同样的条目。  

**env_parent_id**:内核负责给这个字段赋值，它存储了创建这个环境的环境的`env_id`。通过这种方式可以形成一个"family tree"，这样对限制某个环境能对哪些环境执行什么操作是十分有效的。  

**env_type**:常用来区别特殊的环境。大多数情况都是`ENV_TYPE_USER`。在后面的lab中会引进更多的值。  

**env_status**:此变量保存下面的值中的一个：

`ENV_FREE`:表示`Env`结构体是不活动的，因此在`env_free_list`上。  

`ENV_RUNNABLE`:表示`Env`结构体代表一个正在等待运行在CPU中的环境。

`ENV_RUNNING`:表示`Env`当前正在运行的环境。

`ENV_NOT_RUNNABLE`:表示`Env`结构体代表一个正在活动的环境，但目前无法运行，比如，他正在等待来自另一个环境的进程间通信。(IPC)。  

`ENV_DYING`:表示`Env`结构体代表一个僵尸环境。僵尸环境将在下一次陷入内核的时候被释放。我们在lab4前都不会使用这个标志。  

**env_pgdir**:这个变量保存了这个环境页目录的内核虚拟地址。  

就像Unix 进程一样，JOS 环境在概念上包括**线程**和**地址空间**。线程主要由`saved register`的`env_tf`字段定义，地址空间由`env_pgdir`指向的页目录和页表定义。为了运行一个环境，内核必须同时使用`saved register`和合适的地址空间来设置CPU。  

JOS的`struct Env`和xv6中的`struct proc`类似，两个结构体都在`Trapframe`结构体中保存了环境(进程)的用户模式寄存器状态。在JOS中，各个环境不像xv6中的进程那样具有自己的内核堆栈。一次在内核中只能有一个活动的JOS环境，因此JOS只需要一个内核堆栈。  

### Allocating the Environments Array

在lab2中，我们在`mem_init()`中为`pages`数组分配了内存，它是内核用来记录哪些页面是空闲的，哪些不是的一张表。进一步修改`mem_init()`来分配一个相似的`Env`结构体数组`envs`。

> **Exercise 1.** Modify `mem_init()` in `kern/pmap.c` to allocate and map the `envs` array. This array consists of exactly `NENV` instances of the `Env` structure allocated much like how you allocated the `pages` array. Also like the `pages` array, the memory backing `envs` should also be mapped user read-only at `UENVS` (defined in `inc/memlayout.h`) so user processes can read from this array.
>
> You should run your code and make sure `check_kern_pgdir()` succeeds.

```c
	// LAB 3: Your code here.
	envs = (struct Env *) boot_alloc(NENV * sizeof(struct Env));
	memset(envs, 0, NENV * sizeof(struct Env));
```

### Creating and Running Environments

现在我们需要在`kern/env.c`编写必要的代码来运行一个环境。因为我们现在还没有文件系统，所以我们只能设置内核加载一个内嵌在内核里的静态的二进制镜像。JOS将二进制镜像作为一个ELF可执行镜像嵌入在内核里。  

Lab3的`GNUMakefiles`在`obj/user`目录生成了大量的二进制镜像。他们是如何内嵌进内核中是由linker管理的，此处省略。  

在`kern/init.c:i386_init()`包含了如何在一个环境中运行一个二进制镜像的代码。

```c
#if defined(TEST)
	// Don't touch -- used by grading script!
	ENV_CREATE(TEST, ENV_TYPE_USER);
#else
	// Touch all you want.
	ENV_CREATE(user_hello, ENV_TYPE_USER);
#endif // TEST*

	// We only have one user environment for now, so just run it.
	env_run(&envs[0]);
```

但是我们还没有完成建立用户环境的关键函数，我们需要实现他们。

> **Exercise 2.** In the file `env.c`, finish coding the following functions:
>
> - `env_init()`
>
>   Initialize all of the `Env` structures in the `envs` array and add them to the `env_free_list`. Also calls `env_init_percpu`, which configures the segmentation hardware with separate segments for privilege level 0 (kernel) and privilege level 3 (user).
>
> - `env_setup_vm()`
>
>   Allocate a page directory for a new environment and initialize the kernel portion of the new environment's address space.
>
> - `region_alloc()`
>
>   Allocates and maps physical memory for an environment
>
> - `load_icode()`
>
>   You will need to parse an ELF binary image, much like the boot loader already does, and load its contents into the user address space of a new environment.
>
> - `env_create()`
>
>   Allocate an environment with `env_alloc` and call `load_icode` to load an ELF binary into it.
>
> - `env_run()`
>
>   Start a given environment running in user mode.
>
> As you write these functions, you might find the new cprintf verb `%e` useful -- it prints a description corresponding to an error code. For example,
>
> ```
> 	r = -E_NO_MEM;
> 	panic("env_alloc: %e", r);
> ```
>
> will panic with the message "env_alloc: out of memory".

**env_init()**

```c
// Mark all environments in 'envs' as free, set their env_ids to 0,
// and insert them into the env_free_list.
// Make sure the environments are in the free list in the same order
// they are in the envs array (i.e., so that the first call to
// env_alloc() returns envs[0]).
//
void
env_init(void)
{
	// Set up envs array
	// LAB 3: Your code here.
	env_free_list = NULL;
	for (int i = NENV - 1; i >= 0; i--) {	//头插法构建链表，注意保证顺序，envs数组最后一个元素是头结点。
		envs[i].env_id = 0;
		envs[i].env_link = env_free_list;
		env_free_list = &envs[i];
	}

	// Per-CPU part of the initialization
	env_init_percpu();
}
```

**env_setup_vm()**

```c
//
// Initialize the kernel virtual memory layout for environment e.
// Allocate a page directory, set e->env_pgdir accordingly,
// and initialize the kernel portion of the new environment's address space.
// Do NOT (yet) map anything into the user portion
// of the environment's virtual address space.
//
// Returns 0 on success, < 0 on error.  Errors include:
//	-E_NO_MEM if page directory or table could not be allocated.
//
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

	// LAB 3: Your code here.
	p->pp_ref++;
	e->env_pgdir = (pde_t *) page2kva(p);
	//注意每个环境都共享内核页表
	memcpy(e->env_pgdir, kern_pgdir, PGSIZE);
	//for (int i = 0; i < 1024; i++) {
	//	cprintf("pgdir[%d]:%x\n", i, e->env_pgdir[i]);
	//}
	// UVPT maps the env's own page table read-only.
	// Permissions: kernel R, user R
	e->env_pgdir[PDX(UVPT)] = PADDR(e->env_pgdir) | PTE_P | PTE_U;

	return 0;
}
```

**region_alloc()**

说一下详细思路，由于内核对内存的管理是按page为单位的，所以如果len不是page的整数倍，就会向page的整数倍取整对齐。  

如何判断要分配多少页的物理内存？计算起始和终止虚拟地址，一页一页的加，每次分配一个物理页。

```c
//
// Allocate len bytes of physical memory for environment env,
// and map it at virtual address va in the environment's address space.
// Does not zero or otherwise initialize the mapped pages in any way.
// Pages should be writable by user and kernel.
// Panic if any allocation attempt fails.
//
static void
region_alloc(struct Env *e, void *va, size_t len)
{
	// LAB 3: Your code here.
	// (But only if you need it for load_icode.)
	//
	// Hint: It is easier to use region_alloc if the caller can pass
	//   'va' and 'len' values that are not page-aligned.
	//   You should round va down, and round (va + len) up.
	//   (Watch out for corner-cases!)
	void *begin = ROUNDDOWN(va, PGSIZE), *end = ROUNDUP(va+len, PGSIZE);
	while (begin < end) {
		struct PageInfo *pg = page_alloc(0); //分配一个物理页
		if (!pg) {
			panic("region_alloc failed\n");
		}
		page_insert(e->env_pgdir, pg, begin, PTE_W | PTE_U);   //修改e->env_pgdir，建立线性地址begin到物理页pg的映射关系
		begin += PGSIZE;    //更新线性地址
	}
}
```

**load_icode()**  

参考`boot/main.c`，需要注意的点是，在进行加载之前，需要切换成用户态，在完成所有操作之后，切换回内核态。

```c
	struct Elf *ELFHDR = (struct Elf *) binary;
	struct Proghdr *ph;				//Program Header
	int ph_num;						//Program entry number
	if (ELFHDR->e_magic != ELF_MAGIC) {
		panic("binary is not ELF format\n");
	}
	ph = (struct Proghdr *) ((uint8_t *) ELFHDR + ELFHDR->e_phoff);
	ph_num = ELFHDR->e_phnum;

	lcr3(PADDR(e->env_pgdir));			//这步别忘了，虽然到目前位置e->env_pgdir和kern_pgdir除了PDX(UVPT)这一项不同，其他都一样。
										//但是后面会给e->env_pgdir增加映射关系

	for (int i = 0; i < ph_num; i++) {
		if (ph[i].p_type == ELF_PROG_LOAD) {		//只加载LOAD类型的Segment
			region_alloc(e, (void *)ph[i].p_va, ph[i].p_memsz);
			memset((void *)ph[i].p_va, 0, ph[i].p_memsz);		//因为这里需要访问刚分配的内存，所以之前需要切换页目录
			memcpy((void *)ph[i].p_va, binary + ph[i].p_offset, ph[i].p_filesz); //应该有如下关系：ph->p_filesz <= ph->p_memsz。搜索BSS段
		}
	}

	lcr3(PADDR(kern_pgdir));

	e->env_tf.tf_eip = ELFHDR->e_entry;
	// Now map one page for the program's initial stack
	// at virtual address USTACKTOP - PGSIZE.

	// LAB 3: Your code here.
	region_alloc(e, (void *) (USTACKTOP - PGSIZE), PGSIZE);
```

**env_create()**

使用`env_alloc`分配一个环境并调用`load_icode`加载一个ELF二进制.

```c
void
env_create(uint8_t *binary, enum EnvType type)
{
	// LAB 3: Your code here.
	struct Env *e;
	if(env_alloc(&e, 0)) {
		panic("env_alloc failed.\n");
	}

	load_icode(e, binary);
	e->env_type = type;
}
```

**env_run()**

在用户模式下启动运行一个坏境。

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

	// LAB 3: Your code here.
	if (curenv) 
		if (curenv->env_status == ENV_RUNNING)
			curenv->env_status = ENV_RUNNABLE;

	curenv = e;
	curenv->env_status = ENV_RUNNING;
	curenv->env_runs += 1;
	lcr3(PADDR(curenv->env_pgdir));
	
	env_pop_tf(&(curenv->env_tf));
	// panic("env_run not yet implemented");
}
```

下面是用户代码的调用图。

- `start` (`kern/entry.S`)
- `i386_init(kern/init.c)`
  - `cons_init`
  - `mem_init`
  - `env_init`
  - `trap_init` (still incomplete at this point)
  - `env_create`
  - `env_run`
    - `env_pop_tf`

编译内核在QEMU启动它，如果一切顺利，系统会进入用户空间执行`hello`镜像直到它使用`int`指令产生一个系统调用。这是可能会有一些麻烦，因为JOS没有设置硬件来允许用户空间到内核空间的转换，也就是说一旦进入用户空间，系统再也无法返回内核空间，这将是非常糟糕的，内核失去了控制权。当CPU发现没有建立起处理这个系统调用的中断，它就会产生一个保护异常，发现它无法被处理，产生一个双重故障异常，然后发现无法处理双重故障异常，最后因为三重故障而放弃。通常，这时候CPU会复位，系统会重启。这对于传统应用程序来说是十分重要的，对于内核开发十分痛苦。  

如果用gdb进行单步调试，就会发现调用cprintf函数输出hello，最后实际上是调用`sys_cputs`， 对应的汇编指令为`int $0x30`，这个`int`指令是在终端显示字符的系统调用。

### Handling Interrupts and Exceptions

`int $0x30`这个系统调用会死掉：因为一旦CPU进入用户模式，就无法返回内核模式了。我们首先需要实现基本的异常和系统调用处理程序，这样我们才能让内核从用户模式恢复控制权。首先得熟悉x86的中断和异常机制。  

### Basics of Protected Control Transfer

异常和中断两者都是受保护的控制权转移，这导致处理器从用户模式切换到内核模式(CPL = 0), 而不会给用户模式任何干扰内核或其他环境的机会。  

在Intel的术语中，中断和异常是有区别的：  

**中断**：由通常发生在CPU外部的异步事件造成的受保护控制权转移，比如外部设备I/O活动的通知。  

**异常**：是由当前运行代码同步引起的受保护控制权转移，比如除零或无效的内存访问。  

为了确保受保护控制权转移能真正的被保护，CPU的中断、异常机制会设计成不让当前正在运行的代码随意进入内核，例如进入内核的哪个位置、如何进入内核都是精心控制的。在x86，两个机制共同工作提供了这种保护：  

1、**The Interrupt Descriptor Table**. 处理器确保中断和异常只能导致在几个特定的、定义明确的入口点进入内核，这些入口点由内核本身决定，而不是中断或异常发生时运行的代码决定。  

x86允许多达256个不同的中断或者异常入口点进入内核，每个入口点都有不同的中断向量：向量是介于0到255之间的数字。中断向量由中断源决定：不同的设备、错误条件以及对内核的应用程序请求会使用不同的向量生成中断。CPU使用向量作为中断描述符表的索引，该表由内核在内核中的专用内存中设置，就像全局描述符表GDT一样。在IDT的相应条目中，处理器将加载：  

- 要加载到指令指针寄存器(EIP)中的值，该值指向为处理这种异常的内核代码。
- 要加载到代码段寄存器(CS)中的值，在低两位0-1位中包含运行异常处理程序的特权级。（在JOS中，所以的异常都在内核模式下处理，特权级0）.

2、**The Task State Segment**.在中断或异常发生之前，处理器需要一个位置来保存旧的处理器状态，例如处理器调用异常处理程序之前的EIP和CS的原始值，以便异常处理程序以后可以恢复该旧状态并从中断处恢复被中断的代码。但是这个保存区域必须不受用户模式代码的影响，否则，有缺陷的或恶意的用户代码可能会损害内核。  

因此，当x86处理器执行中断或陷阱，从而导致特权级别从用户模式更改为内核模式时，它还会切换到内核内存中的堆栈。称为任务状态段（TSS）的结构指定了段选择器和该堆栈所在的地址。处理器（在此新堆栈上）压入SS，ESP，EFLAGS，CS，EIP和可选的错误代码。然后从中断描述符加载CS和EIP，并将ESP和SS设置为引用新堆栈。  

尽管TSS很大，可以潜在地用于多种用途，但JOS仅使用它来定义处理器从用户模式转换到内核模式时应切换到的内核堆栈。由于JOS中的“内核模式”在x86上的特权级别为0，因此处理器在进入内核模式时使用TSS的ESP0和SS0字段来定义内核堆栈。 JOS不使用任何其他TSS字段。  

### Types of Exceptions and Interrupts

x86处理器下所有可以生成的内部同步异常使用0-31号中断向量，映射到IDT0-31号条目。例如，缺页故障总是通过向量14导致异常。大于31的中断向量号只用于软件中断，当外部设备需要注意时，由`int`指令或者异步硬件中断引起。

### An Example

让我们来把这些东西整合到一个例子。处理器正在用户模式下执行代码，这时候遇到了一个尝试除0的除法指令。

1. 处理器切换到由`TSS`字段`SS0`和`ESP0`定义的栈段，在JOS中它们各自保存着`GD_KD`和`KSTACKTOP`.

2. 处理器向`内核栈`中压入异常参数，从`KSTACKTOP`地址开始。

   ```
                        +--------------------+ KSTACKTOP             
                        | 0x00000 | old SS   |     " - 4
                        |      old ESP       |     " - 8
                        |     old EFLAGS     |     " - 12
                        | 0x00000 | old CS   |     " - 16
                        |      old EIP       |     " - 20 <---- ESP 
                        +--------------------+             
   ```

3. 因为我们正在处理一个除0错误，在x86中是中断向量0，处理器读取IDT条目0然后设置`CS:EIP`指向由条目描述的处理程序地址。

4. 处理程序接过控制权开始处理异常，例如终结掉用户环境。

对于x86的某些类型异常，除了上面标准的五字参数以外，处理器还会压入另一个包含`错误码`的字。例如缺页异常，14号，就是一个重要的例子。当处理器压入一个错误码，当从用户模式进入异常处理程序时，堆栈如下所示：  

```
                     +--------------------+ KSTACKTOP             
                     | 0x00000 | old SS   |     " - 4
                     |      old ESP       |     " - 8
                     |     old EFLAGS     |     " - 12
                     | 0x00000 | old CS   |     " - 16
                     |      old EIP       |     " - 20
                     |     error code     |     " - 24 <---- ESP
                     +--------------------+             
	
```

### Nested Exceptions and Interrupts

处理器可以接受来自内核和用户模式下的异常和中断。但是，只有从用户模式进入内核时，x86处理器才会自动切换堆栈，然后再将其旧的寄存器状态推入堆栈并通过IDT调用适当的异常处理程序。如果在发生中断或异常时处理器已经处于内核模式（CS寄存器的低2位已经为零）。然后CPU会将更多的值压入同一内核堆栈。这样，内核可以优雅地处理由内核本身内的代码引起的嵌套异常。该功能是实现保护的重要工具，我们将在后面的系统调用部分中看到。  

如果处理器已经处于内核模式下并采用嵌套异常，由于不需要切换堆栈，因此不会保存旧的SS或ESP寄存器。因此，对于不推送错误代码的异常类型，内核堆栈在进入异常处理程序时看起来类似于以下内容：

```
                     +--------------------+ <---- old ESP
                     |     old EFLAGS     |     " - 4
                     | 0x00000 | old CS   |     " - 8
                     |      old EIP       |     " - 12
                     +--------------------+             
```

对于推送错误代码的异常类型，处理器会像以前一样在旧EIP之后立即推送错误代码。  

处理器的嵌套异常功能有一个重要警告：如果处理器在已经处于内核模式下时发生异常，并且由于诸如堆栈空间不足之类的任何原因而无法将其旧状态推入内核堆栈，那么处理器就无能为力，只能恢复自身状态。不用说，应该对内核进行设计，以免发生这种情况。

### Setting Up the IDT

现在，您应该具有设置IDT和处理JOS中的异常所需的基本信息。您将设置IDT来处理中断向量0-31（处理器异常）。在本实验的后面，我们将处理系统调用中断，并在以后的实验中添加中断32-47（设备IRQ）。  

头文件`inc/trap.h`和`kern/trap.h`包含与您需要熟悉的中断和异常相关的重要定义。文件`kern/trap.h`包含对内核严格专有的定义，而`inc/trap.h`包含对用户级程序和库也可能有用的定义。  

注意：Intel定义0-31范围内的某些例外情况为保留。由于它们永远不会由处理器生成，因此如何处理它们并不重要。做您认为最干净的事情。  

您应该实现的总体控制流程如下所示：

```
      IDT                   trapentry.S         trap.c
   
+----------------+                        
|   &handler1    |---------> handler1:          trap (struct Trapframe *tf)
|                |             // do stuff      {
|                |             call trap          // handle the exception/interrupt
|                |             // ...           }
+----------------+
|   &handler2    |--------> handler2:
|                |            // do stuff
|                |            call trap
|                |            // ...
+----------------+
       .
       .
       .
+----------------+
|   &handlerX    |--------> handlerX:
|                |             // do stuff
|                |             call trap
|                |             // ...
+----------------+
```

每个异常或中断在`trapentry.S`中有自己独有的处理程序，`trap_init()`应该初始化用这些处理程序初始化IDT。每个处理程序都应该在栈上构造一个`struct Trapframe(see inc/trap.h)`然后使用指向`Trapframe`的指针调用`trap()`（in `trap.c`）。然后，trap（）处理异常/中断或将其分派到特定的处理函数。

> **Exercise 4.** Edit `trapentry.S` and `trap.c` and implement the features described above. The macros `TRAPHANDLER` and `TRAPHANDLER_NOEC` in `trapentry.S` should help you, as well as the T_* defines in `inc/trap.h`. You will need to add an entry point in `trapentry.S` (using those macros) for each trap defined in `inc/trap.h`, and you'll have to provide `_alltraps` which the `TRAPHANDLER` macros refer to. You will also need to modify `trap_init()` to initialize the `idt` to point to each of these entry points defined in `trapentry.S`; the `SETGATE` macro will be helpful here.
>
> Your `_alltraps` should:
>
> 1. push values to make the stack look like a struct Trapframe
> 2. load `GD_KD` into `%ds` and `%es`
> 3. `pushl %esp` to pass a pointer to the Trapframe as an argument to trap()
> 4. `call trap` (can `trap` ever return?)
>
> Consider using the `pushal` instruction; it fits nicely with the layout of the `struct Trapframe`.
>
> Test your trap handling code using some of the test programs in the `user` directory that cause exceptions before making any system calls, such as `user/divzero`. You should be able to get make grade to succeed on the `divzero`, `softint`, and `badsegment` tests at this point.

```assembly
; trapentry.S
/*
 * Lab 3: Your code here for generating entry points for the different traps.
 */
	; it will expand to the code
	; pay attension to which exception have error code and which don't 
	TRAPHANDLER_NOEC(th0, 0)
	TRAPHANDLER_NOEC(th1, 1)
	TRAPHANDLER_NOEC(th3, 3)
	TRAPHANDLER_NOEC(th4, 4)
	TRAPHANDLER_NOEC(th5, 5)
	TRAPHANDLER_NOEC(th6, 6)
	TRAPHANDLER_NOEC(th7, 7)
	TRAPHANDLER(th8, 8)
	TRAPHANDLER_NOEC(th9, 9)
	TRAPHANDLER(th10, 10)
	TRAPHANDLER(th11, 11)
	TRAPHANDLER(th12, 12)
	TRAPHANDLER(th13, 13)
	TRAPHANDLER(th14, 14)
	TRAPHANDLER_NOEC(th16, 16)

/*
 * Lab 3: Your code here for _alltraps
 */
_alltraps:
	pushl %ds
	pushl %es
	pushal
	movl $GD_KD, %eax
	movl %eax, %ds
	movl %eax, %es
	pushl %esp
	call trap

```

```c
void
trap_init(void)
{
	extern struct Segdesc gdt[];

	// LAB 3: Your code here.
	void th0();
	void th1();
	void th3();
	void th4();
	void th5();
	void th6();
	void th7();
	void th8();
	void th9();
	void th10();
	void th11();
	void th12();
	void th13();
	void th14();
	void th16();
	SETGATE(idt[0], 0, GD_KT, th0, 0);		//格式如下：SETGATE(gate, istrap, sel, off, dpl)，定义在inc/mmu.h中
	SETGATE(idt[1], 0, GD_KT, th1, 0);  //设置idt[1]，段选择子为内核代码段，段内偏移为th1
	SETGATE(idt[3], 0, GD_KT, th3, 3);
	SETGATE(idt[4], 0, GD_KT, th4, 0);
	SETGATE(idt[5], 0, GD_KT, th5, 0);
	SETGATE(idt[6], 0, GD_KT, th6, 0);
	SETGATE(idt[7], 0, GD_KT, th7, 0);
	SETGATE(idt[8], 0, GD_KT, th8, 0);
	SETGATE(idt[9], 0, GD_KT, th9, 0);
	SETGATE(idt[10], 0, GD_KT, th10, 0);
	SETGATE(idt[11], 0, GD_KT, th11, 0);
	SETGATE(idt[12], 0, GD_KT, th12, 0);
	SETGATE(idt[13], 0, GD_KT, th13, 0);
	SETGATE(idt[14], 0, GD_KT, th14, 0);
	SETGATE(idt[16], 0, GD_KT, th16, 0);

	// Per-CPU setup 
	trap_init_percpu();
}
```

