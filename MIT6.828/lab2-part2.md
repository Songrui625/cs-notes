---
title: 'xv6-6.828-lab2-part2:Virtual Memory'
date: 2021-02-02 00:01:01
tags: xv6

---

在开始这个part之前，需要熟悉x86保护模式下的内存管理机制：分段和页面翻译。

<!--more-->

> **Exercise 2.** Look at chapters 5 and 6 of the [Intel 80386 Reference Manual](https://pdos.csail.mit.edu/6.828/2018/readings/i386/toc.htm), if you haven't done so already. Read the sections about page translation and page-based protection closely (5.2 and 6.4). We recommend that you also skim the sections about segmentation; while JOS uses the paging hardware for virtual memory and protection, segment translation and segment-based protection cannot be disabled on the x86, so you will need a basic understanding of it.

**一定要去看80386的参考手册，一定要看5.2节和6.4节**

## Virtual, Linear, and Physical Addresses

在**x86架构**术语里，虚拟地址(`virtual address`)包括一个段选择器(`segment selector`)和一个偏移量(`offset`)。线性地址(`linear address`)是在段翻译之后得到的。物理地址(`physical address`)是在页翻译后得到的，最后会在总线上传到RAM。  

注意：**x86架构(32位)**包含虚拟地址、线性地址和物理地址这三个术语。而linux内核中明确说明了：**虚拟地址就是线性地址，因为禁用了分段机制**。而在术语上通常说线性地址，但在内核编写中，通常说的是虚拟地址，这是一个历史包袱，不用过于纠结。  

C指针是虚拟地址的`offset`部分，在`boot/boot.S`中，我们**建立了`全局描述符表(GDT)`，它可以通过设置所有的段基地址到0，范围到0xffffffff来关闭分段翻译机制**。因此段选择器不会生效而且线性地址总是等于虚拟地址的偏移量，也就是说我们可以不去考虑线性地址，甚至不用去考虑段选择器，直接将指针看作线性地址就行，默认虚拟地址等于线性地址。在lab3中会与分段进行更多的打交道来设置特权级。但至于内存翻译，在所有labs中，我们可以忽略分段而专注于分页。  

回想下lab1的part3，我们建立了一个简单的页表，这样内核可以在链接地址0xf0100000上运行代码，即使它实际上是被加载到物理内存ROM BIOS上地址为0x00100000。这个页表仅仅映射了4MB的的内存。在这个lab中，我们会做一下拓展，映射从虚拟地址0xf0000000开始的256MB字节内存，并且映射虚拟地址空间中的许多其他区域。  

一旦进入到实模式，就无法直接使用线性或者物理地址，所有的内存引用都会被解释为虚拟地址然后被MMU进行翻译，这意味着**C语言中所有的指针都是虚拟地址**。  

JOS 内核定义了两种地址`uintptr_t：虚拟地址`和`phyaddr_t:物理地址`，它们都是`uint32_t`的同义词，意味着它们都是整数，不是指针，不可以被解引用，但是强制转换为指针后是可以解引用的。物理地址解引用后得到的地址并不能真正的向其读和写，等等就会介绍为什么。

> **Question1** Assuming that the following JOS kernel code is correct, what type should variable `x` have, `uintptr_t` or `physaddr_t`?
>
> ```c
> 	mystery_t x;
> 	char* value = return_a_pointer();
> 	*value = 10;
> 	x = (mystery_t) value;
> ```

这题应该已经可以一秒钟回答了，因为C语言中指针都是虚拟地址。接下来介绍两种转换：1、物理地址转换为虚拟地址（以及为什么）。2、虚拟地址转换为物理地址。  

有时内核需要修改或读取或者修改它仅知道的物理地址的内存。比如，向页表添加映射可能需要分配物理内存来存储`页目录(page directory)`，然后初始化该内存。但是内核是无法绕过虚拟地址翻译这一环节的，也就是说你无法直接从物理地址处的内存读和写数据。那怎么才能实现向已知的物理地址内存读和写数据呢，必须通过虚拟地址，lab提供了一些有用的宏帮助我们进行**物理地址向虚拟地址的映射**。JOS将所有从0开始的物理地址映射到虚拟地址0xf0000000，这是为了在仅知道物理情况下帮助内核读写物理地址对应内存（映射之后，通过MMU将虚拟地址再翻译成物理地址传到总线上读写内存，虽有有点啰嗦，但是必须这么做）。物理内存转换为虚拟地址具体方法就是：内核的物理地址必须加上0xf0000000，这里其实是线性映射，y(x) = x + 0xf0000000, x就是内核的物理地址，y就是对应的虚拟地址。你也可以通过`宏KADDR(pa)`进行这种加法运算。  

内核有时又需要在仅知道虚拟地址（存储在某种内核数据结构）的情况下找到它的对应的物理地址，`boot_alloc()`分配的全局变量和内存位于从0xf0000000开始的内核加载区域中，就是我们映射的所有的物理内存区域。因此，要将该区域中的虚拟地址转换为物理地址，只需要做相反的操作：将虚拟地址减去0xf0000000，就可以得到对应的物理地址，只是和物理地址映射到虚拟地址相反的操作。你可以使用`宏PADDR(va)`进行这种减法运算。

## Reference counting

接下来的lab你会经常在多个虚拟地址上同时映射相同的物理页，你需要维护对应每个物理页的结构体`struct PageInfo`中的一个字段`pp_ref`的引用计数值。当一个物理页`pp_ref`的值为0时，该页面就可以被释放了，因为不会再被使用到（和GC中的引用计数思想一样），通常这个计数值应该等于所有页表中物理页出现在`UTOP`下方的次数，因为`UTOP`上方的映射大多数是在内核启动时设置的，永远都不会被释放，所以无需引用计数。`pp_ref`还用来跟踪指向页目录页面指针的数量。  

当使用`page_alloc`函数时务必当心，它的返回页面的计数值永远为0，所以在你操作完返回的页面后（比如将这页插入一个页表），`pp_ref`的值应该尽快的增加，有时候这是由其他函数操作的（比如，`page_insert`），而有时调用`page_alloc`必须直接增加计数。  

## Page Table Management

现在，你将编写一系列函数来管理页表，插入或者删除 线性到物理地址 的映射， 在必要时创建页表等等功能。在开始这前，我想聊一下虚拟内存, 下面这张图困扰了我许久。

```c

/*
 * Virtual memory map:                                Permissions
 *                                                    kernel/user
 *
 *    4 Gig -------->  +------------------------------+
 *                     |                              | RW/--
 *                     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 *                     :              .               :
 *                     :              .               :
 *                     :              .               :
 *                     |~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~| RW/--
 *                     |                              | RW/--
 *                     |   Remapped Physical Memory   | RW/--
 *                     |                              | RW/--
 *    KERNBASE, ---->  +------------------------------+ 0xf0000000      --+
 *    KSTACKTOP        |     CPU0's Kernel Stack      | RW/--  KSTKSIZE   |
 *                     | - - - - - - - - - - - - - - -|                   |
 *                     |      Invalid Memory (*)      | --/--  KSTKGAP    |
 *                     +------------------------------+                   |
 *                     |     CPU1's Kernel Stack      | RW/--  KSTKSIZE   |
 *                     | - - - - - - - - - - - - - - -|                 PTSIZE
 *                     |      Invalid Memory (*)      | --/--  KSTKGAP    |
 *                     +------------------------------+                   |
 *                     :              .               :                   |
 *                     :              .               :                   |
 *    MMIOLIM ------>  +------------------------------+ 0xefc00000      --+
 *                     |       Memory-mapped I/O      | RW/--  PTSIZE
 * ULIM, MMIOBASE -->  +------------------------------+ 0xef800000
 *                     |  Cur. Page Table (User R-)   | R-/R-  PTSIZE
 *    UVPT      ---->  +------------------------------+ 0xef400000
 *                     |          RO PAGES            | R-/R-  PTSIZE
 *    UPAGES    ---->  +------------------------------+ 0xef000000
 *                     |           RO ENVS            | R-/R-  PTSIZE
 * UTOP,UENVS ------>  +------------------------------+ 0xeec00000
 * UXSTACKTOP -/       |     User Exception Stack     | RW/RW  PGSIZE
 *                     +------------------------------+ 0xeebff000
 *                     |       Empty Memory (*)       | --/--  PGSIZE
 *    USTACKTOP  --->  +------------------------------+ 0xeebfe000
 *                     |      Normal User Stack       | RW/RW  PGSIZE
 *                     +------------------------------+ 0xeebfd000
 *                     |                              |
 *                     |                              |
 *                     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 *                     .                              .
 *                     .                              .
 *                     .                              .
 *                     |~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~|
 *                     |     Program Data & Heap      |
 *    UTEXT -------->  +------------------------------+ 0x00800000
 *    PFTEMP ------->  |       Empty Memory (*)       |        PTSIZE
 *                     |                              |
 *    UTEMP -------->  +------------------------------+ 0x00400000      --+
 *                     |       Empty Memory (*)       |                   |
 *                     | - - - - - - - - - - - - - - -|                   |
 *                     |  User STAB Data (optional)   |                 PTSIZE
 *    USTABDATA ---->  +------------------------------+ 0x00200000        |
 *                     |       Empty Memory (*)       |                   |
 *    0 ------------>  +------------------------------+                 --+
 *
 * (*) Note: The kernel ensures that "Invalid Memory" is *never* mapped.
 *     "Empty Memory" is normally unmapped, but user programs may map pages
 *     there if desired.  JOS user programs map pages temporarily at UTEMP.
 */
```

这一张图看起来是不是像一个内存图，某个地址存着一个对象，这个认知浪费了我很多时间，最重要的是这个理解是错误的：虚拟地址并不可以直接存放对象，那上面这张图是什么意思呢？  

这张虚拟内存布局其实并没有什么问题，讲义里强调过地址映射，虚拟内存是通过映射转换到物理内存的，但是这是MMU负责的，MMU通过查找页表来进行映射，那我们这里能不能做类似的事情，让虚拟地址映射到一个物理内存呢？  

当然可以，我们只需要让**虚拟地址存放指向对象的指针**就可以表现出上图那样，虚拟地址存放对象，但是页表是保存虚拟地址到物理地址的映射，所以**虚拟地址应该存放对象的物理地址，而不是指向对象的指针，因为指针是个虚拟地址**，举个例子来说明下。

```c
//页表保存虚拟地址到物理地址的映射，虚拟地址UPAGES保存的对象是RO PAGES，前面提到过实际上是虚拟地址UPAGES保存的是指向对象的指针，也就是pages数组的指针。
//我们通过虚拟地址计算出页目录项的索引、页表项的索引、以及偏移量。
//UPAGES:ef000000 --->  11101111000000000000000000000000
//高10位：1110111100  次10位：0000000000 低12位：000000000000
//页目录项idx:956     页表项idx:0        偏移量:0
//如果没有二级页表，那么此时kern_pgdir[956] = pages的物理地址
//但是有二级页表机制，所以kern_pgdir[956] = pt_x(页表的物理地址)
//如果页表不存在，那我们就给分配一页物理页给页表，一共1024项页表项的数组。
//而且我们有了页表项索引，那就可以得到了：*(pt_x + idx) = pages的物理地址。
//这样我们就在页表上保存了一组虚拟地址到物理地址的映射
```

所以说，上面那个虚拟内存布局图其实就是 虚拟内存映射图，指示了每一组虚拟地址到物理地址的映射。  

现在来总结下如何向页表中插入一组虚拟地址到物理地址的映射：  

1、根据虚拟地址va计算出页目录索引pdx和页表索引ptx.

2、如果页表为空那就分配一个新的物理页。

3、在页表项上插入待映射的物理地址。

> **Exercise 4.** In the file `kern/pmap.c`, you must implement code for the following functions.
>
> ```c
> pgdir_walk()
> boot_map_region()
> page_lookup()
> page_remove()
> page_insert()
> ```
>
> `check_page()`, called from `mem_init()`, tests your page table management routines. You should make sure it reports success before proceeding.

**pgdir_walk()**:

参数:1.pgdir: 页目录的虚拟地址 2.va: 虚拟地址 3.create: 页表为空时是否创建页表，true为创建，false为不创建。

返回值:指向va对应页表项(entry可以翻译为条目或者项)的指针。  

在x86中，虚拟地址va的格式与翻译过程如下：

![image-20210203124318815](.\figure\image-20210203124318815.png)

![image-20210203124145016](.\figure\image-20210203124145016.png)

![image-20210204111129901](.\figure\image-20210204111129901.png?lastModify=1612412610)

下面来讲解一下Page Trasnlation的过程，首先通过虚拟地址的高10位拿到页目录的索引，这样就可以定位到页目录项。有了页目录项后，可以通过查看权限位Present来判断页面是否生效，如果生效，我们就可以通过页目录项的高20位拿到页表的物理地址，然后我们通过虚拟地址的中间10位拿到页表的索引，这样就可以定位到页表项，继续查看权限位Present来判断该页是否生效，如果生效，我们就可以通过页表项的高20位拿到具体页的物理地址。最后通过虚拟地址的低12位拿到页的索引，这样我们就成功拿到了对应的物理地址了。  

为什么需要页目录，单独页表不好吗？页目录可以提高寻址范围，页目录中有1024个页目录项，页表中有1024个个页表项，最大可寻址1024*1024 = 1M个页。如果只是单独一个页表，最大可寻址1024个页。  

页目录项和页表项中的页帧地址是物理地址还是虚拟地址：页帧指的是物理地址。  

理解了以上两张图的内容后，还需要了解`mmu.h`头文件中的几个宏:`PDX(la)`, `PTX(la)`, `PTE_ADDR(pte)`, `PTE_U`, `PTE_W`, `PTE_P`，以及头文件`pmap.h`中的函数`page2pa`. 需要提一下的是，在JOS中虚拟地址va可以看作于线性地址la，前文已经描述过了，`mmu.h`中也有提及。

```c
// Given 'pgdir', a pointer to a page directory, pgdir_walk returns
// a pointer to the page table entry (PTE) for linear address 'va'.
// This requires walking the two-level page table structure.
//
// The relevant page table page might not exist yet.
// If this is true, and create == false, then pgdir_walk returns NULL.
// Otherwise, pgdir_walk allocates a new page table page with page_alloc.
//    - If the allocation fails, pgdir_walk returns NULL.
//    - Otherwise, the new page's reference count is incremented,
//	the page is cleared,
//	and pgdir_walk returns a pointer into the new page table page.
//
// Hint 3: look at inc/mmu.h for useful macros that manipulate page
// table and page directory entries.
//
pte_t *
pgdir_walk(pde_t *pgdir, const void *va, int create)
{
	// Fill this function in
	size_t pdx;
	size_t ptx;
	pte_t *pde;
	struct PageInfo *pp;
	pdx = PDX(va);
	ptx = PTX(va);
	pde = pgdir + pdx;

	//page table page not exist
	if (!(*pde & PTE_P)) {
		if (create == false) return NULL;
		pp = page_alloc(ALLOC_ZERO);
		if (!pp) return NULL;
		pp->pp_ref++;
		*pde = page2pa(pp) | PTE_P | PTE_U | PTE_W; //保留足够多的权限，用户可读写，并且存在。
	}

	return (pte_t *) KADDR(PTE_ADDR(*pde)) + ptx;
}
```

通过struct PageInfo结构体的注释可以知道，struct PageInfo与物理页帧是一一对应的关系，但并不是物理页本身，但可以通过`page2pa()`拿到页的物理地址，这里需要注意的是物理地址已经是32位了，高20位是物理地址，低12位为0。  

最后的返回语句，`*pde`值为页目录项中对应页表的的物理地址+权限位，`PTE_ADDR(*pde)`则是取出高20位，也就是页表页物理地址。`KADDR(PTE_ADDR(*pde))`则是返回页表页物理地址对应的虚拟地址，然后强制转换为`pte_t *`，这样就得到了页表数组，在加上`PTX(va)`就定位到了虚拟地址对应的页表项的虚拟地址，返回它。  

为什么是返回一个虚拟地址呢？因为MMU是集成在硬件中的，x86是无法绕过MMU的，所以直接返回物理地址是无法使用的，也就是说如果返回了页表项的物理地址，那么MMU又会再一次将这个物理地址当成虚拟地址进行translate，这样得到的地址就不是我们想要的了。而且注释也说了返回一个指针，说明是返回虚拟地址。  

**boot_map_region()**  

参数：1、指向页目录的指针:pgdir。2、虚拟地址:va。3、映射的字节大小：size。4、物理地址：pa。5、权限：perm。  

```c
//
// Map [va, va+size) of virtual address space to physical [pa, pa+size)
// in the page table rooted at pgdir.  Size is a multiple of PGSIZE, and
// va and pa are both page-aligned.
// Use permission bits perm|PTE_P for the entries.
//函数的作用是在根页目录的页表中，将虚拟地址空间[va, va + size)的内存映射到物理地址[pa, pa + size)处。`size`是PGSIZE的整数倍，而且`va` 和`pa`必须是页面对齐的。  

//该函数只期望建立起位于UTOP之上的静态映射。因此，不应该改变`pp_ref`字段。  
// This function is only intended to set up the ``static'' mappings
// above UTOP. As such, it should *not* change the pp_ref field on the
// mapped pages.
//
// Hint: the TA solution uses pgdir_walk
static void
boot_map_region(pde_t *pgdir, uintptr_t va, size_t size, physaddr_t pa, int perm)
{
	// Fill this function in
	size_t i;
	for (i = 0; i < size / PGSIZE; i++) {
		pte_t *pte = pgdir_walk(pgdir, (void *) va + i*PGSIZE, 1);	//create
		if (!pte) panic("boot_map_region() panic:out of memory");
		*pte = (pa + i*PGSIZE) | perm | PTE_P;
	}
}
```

注意，内存管理都是以4KB(页大小)为最小单元进行管理的。由于size是PGSIZE的整数倍，所以一共要映射 size / PGSIZE 页，对每一页，拿到它的页表项地址，同时映射到相应的物理地址，这回覆盖掉原来的映射。(**这里我的疑惑时，如果页表项为空，那么在pgdir_walk中就会创建一个新的页，并且映射到新的页，那这里为什么还要覆盖掉旧的映射？是不是有点多此一举？**)  

**page_lookup()**  

参数：1、页目录pgdir。2、虚拟地址va。3、需要存储的页表项 pte_store。  

返回值：映射在虚拟地址va的页的对应物理结构PageInfo指针。  

```c
//
// Return the page mapped at virtual address 'va'.
// If pte_store is not zero, then we store in it the address
// of the pte for this page.  This is used by page_remove and
// can be used to verify page permissions for syscall arguments,
// but should not be used by most callers.
// 返回映射在虚拟地址va的物理页，并且将这页的页表项pte存储在pte_store中。
//
// Return NULL if there is no page mapped at va.
// 
// Hint: the TA solution uses pgdir_walk and pa2page.
// 
struct PageInfo *
page_lookup(pde_t *pgdir, void *va, pte_t **pte_store)
{
	// Fill this function in
	pte_t *pte;
	struct PageInfo *pp;
	pte  = pgdir_walk(pgdir, va, 0);
	if (!pte || !(*pte & PTE_P)) return NULL; // 检查页面是否存在，只检查pte为空时不够的，一定要包括权限检查。
	// pp = pg2page(*pte) 是错误的，这个物理地址包括了权限位，并不是物理地址。
	// 要通过PTE_ADDR拿到页表项的物理地址，其实就是把低12位置0
	pp = pa2page(PTE_ADDR(*pte));
	if (pte_store) *pte_store = pte;
	return pp;
}
```

**page_remove()**  

参数：1、页目录pde。 2、虚拟地址va。  

```c

//
// Unmaps the physical page at virtual address 'va'.
// If there is no physical page at that address, silently does nothing.
// 在页表中移除虚拟地址va的映射, 如果页表项没有被映射，则什么也不用做。
// Details:
//   - The ref count on the physical page should decrement.
//   - The physical page should be freed if the refcount reaches 0.
//   - The pg table entry corresponding to 'va' should be set to 0.
//     (if such a PTE exists)
//   - The TLB must be invalidated if you remove an entry from
//     the page table.
//
// Hint: The TA solution is implemented using page_lookup,
// 	tlb_invalidate, and page_decref.
//
void
page_remove(pde_t *pgdir, void *va)
{
	// Fill this function in
	pte_t *pte;
	struct PageInfo *pp;
	pp = page_lookup(pgdir, va, &pte);
	if (!pp || !(*pte & PTE_P)) return;
	page_decref(pp);
	// if (!pp->pp_ref) page_free(pp); //错误的写法，page_decref已经包含了
	*pte = 0;
	tlb_invalidate(pgdir, va);
}
```

注释有一句话，误导我了。page_decref已经包含了对ref为0的检查。  

**page_insert**

参数：1、页目录pgdir。2、物理页状态对应结构 struct PageInfo *pp。 3、权限 perm。  

返回值：0代表插入成功， 1代表插入失败。  

```c
//
// Map the physical page 'pp' at virtual address 'va'.
// The permissions (the low 12 bits) of the page table entry
// should be set to 'perm|PTE_P'.
// 映射物理页 ‘pp’ 到虚拟地址 va，同时设置权限为 perm | PTE_P
// Requirements
//   - If there is already a page mapped at 'va', it should be page_remove()d.
//     如果虚拟地址va已经有映射，那么旧调用page_remove()把它移除。
//   - If necessary, on demand, a page table should be allocated and inserted
//     into 'pgdir'.
//     如果有必要，可以分配一个页表然后插入到页目录中。
//   - pp->pp_ref should be incremented if the insertion succeeds.
//     如果插入成功，那么引用计数字段应该加一。
//   - The TLB must be invalidated if a page was formerly present at 'va'.
//     如果va之前已经映射一页，快表应该被校验。
//
// Corner-case hint: Make sure to consider what happens when the same
// pp is re-inserted at the same virtual address in the same pgdir.
// However, try not to distinguish this case in your code, as this
// frequently leads to subtle bugs; there's an elegant way to handle
// everything in one code path.
//
// RETURNS:
//   0 on success
//   -E_NO_MEM, if page table couldn't be allocated
//
// Hint: The TA solution is implemented using pgdir_walk, page_remove,
// and page2pa.
//
int
page_insert(pde_t *pgdir, struct PageInfo *pp, void *va, int perm)
{
	// Fill this function in
	pte_t *pte;
	// pte = pgdir_walk(pgdir, va, 0); //如果页表项不存在要创建页表项，为什么？因为一定要插入va->pa的地址映射。
	pte = pgdir_walk(pgdir, va, 1);   //如果没有空闲页面用于创建页表，那么内存不足，无法插入。
	if (!pte) return -E_NO_MEM;

	pp->pp_ref++;
	if (*pte & PTE_P) page_remove(pgdir, va);
	*pte = page2pa(pp) | perm | PTE_P;

	return 0; 
}
```



