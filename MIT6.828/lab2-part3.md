JOS把处理器32位的线性地址空间分成两部分。用户空间（进程）,它负责控制控制下部的布局和内容，而内核空间始终保持对上部的完全控制。<!--more-->分界线是由`inc/memlayout.h`中的`ULIM`定义的，为内核保留大约256M的虚拟地址空间。这样也解释了为什么我们在lab1中将内核安排在这么高的一个链接地址，否则，在内核的虚拟地址空间中将没有足够的空间同时映射到其下方的用户环境中。  

## Permission and Fault Isolation

因为内核和用户内存都存在于每个环境的地址空间中，我们不得不在x86页表中使用权限位来允许用户代码只可以访问地址空间的用户部分。否则，用户代码的bugs可能会覆盖内核数据，导致崩溃或者更奇妙的故障。用户代码也可以窃取其他环境的私有数据。注意到可写权限位(`PTE_W`)会同时影响用户代码和内核代码。  

对于`ULIM`上方的内存，用户环境没有任何权限，但是内核可能会读写这部分内存。对于`[UTOP, ULIM)`范围内的地址，内核和用户环境拥有同样的权限：可以读取但不能写入。这个范围的地址常用于将某些内核数据结构以只读方式暴露给用户环境。最后，`UTOP`以下的地址空间供用户环境使用。用户环境会设置权限来访问这部分内存。  

## Initializing the Kernel Address Space

现在你将建立`UTOP`以上的地址空间：地址空间的内核部分。(这句话有一些问题，既然`ULIM`才是内核空间和用户空间的分界线，为什么这里说的是`UTOP`以上？)`inc/memlayout.h`展示了你将使用的内存布局。你将会使用你刚刚编写的那些函数来建立线性地址到物理地址的映射。  

> **Exercise 5.** Fill in the missing code in `mem_init()` after the call to `check_page()`.
>
> Your code should now pass the `check_kern_pgdir()` and `check_page_installed_pgdir()` checks.

```c

	//////////////////////////////////////////////////////////////////////
	// Now we set up virtual memory

	//////////////////////////////////////////////////////////////////////
	// Map 'pages' read-only by the user at linear address UPAGES
	// Permissions:
	//    - the new image at UPAGES -- kernel R, user R
	//      (ie. perm = PTE_U | PTE_P)
	//    - pages itself -- kernel RW, user NONE
	// Your code goes here:
	boot_map_region(kern_pgdir, UPAGES, PTSIZE, PADDR(pages), PTE_U);
	//////////////////////////////////////////////////////////////////////
	// Use the physical memory that 'bootstack' refers to as the kernel
	// stack.  The kernel stack grows down from virtual address KSTACKTOP.
	// We consider the entire range from [KSTACKTOP-PTSIZE, KSTACKTOP)
	// to be the kernel stack, but break this into two pieces:
	//     * [KSTACKTOP-KSTKSIZE, KSTACKTOP) -- backed by physical memory
	//     * [KSTACKTOP-PTSIZE, KSTACKTOP-KSTKSIZE) -- not backed; so if
	//       the kernel overflows its stack, it will fault rather than
	//       overwrite memory.  Known as a "guard page".
	//     Permissions: kernel RW, user NONE
	// Your code goes here:
	boot_map_region(kern_pgdir, KSTACKTOP - KSTKSIZE, KSTKSIZE, PADDR(bootstack), PTE_W);
	//////////////////////////////////////////////////////////////////////
	// Map all of physical memory at KERNBASE.
	// Ie.  the VA range [KERNBASE, 2^32) should map to
	//      the PA range [0, 2^32 - KERNBASE)
	// We might not have 2^32 - KERNBASE bytes of physical memory, but
	// we just set up the mapping anyway.
	// Permissions: kernel RW, user NONE
	// Your code goes here:
	size_t kern_size = ROUNDUP((0xffffffff - KERNBASE), PGSIZE);
	boot_map_region(kern_pgdir, KERNBASE, kern_size, PADDR((void *)KERNBASE), PTE_W);
```

注意设置相应的权限就可以了，没有什么太大的问题。  

> **Question**
>
> 1. What entries (rows) in the page directory have been filled in at this point? What addresses do they map and where do they point? In other words, fill out this table as much as possible:
>
>    | Entry | Base Virtual Address | Points to (logically):                |
>    | ----- | -------------------- | ------------------------------------- |
>    | 1023  | ?                    | Page table for top 4MB of phys memory |
>    | 1022  | ?                    | ?                                     |
>    | .     | ?                    | ?                                     |
>    | .     | ?                    | ?                                     |
>    | .     | ?                    | ?                                     |
>    | 2     | 0x00800000           | ?                                     |
>    | 1     | 0x00400000           | ?                                     |
>    | 0     | 0x00000000           | [see next question]                   |
>
> 2. We have placed the kernel and user environment in the same address space. Why will user programs not be able to read or write the kernel's memory? What specific mechanisms protect the kernel memory?
>
> 3. What is the maximum amount of physical memory that this operating system can support? Why?
>
> 4. How much space overhead is there for managing memory, if we actually had the maximum amount of physical memory? How is this overhead broken down?
>
> 5. Revisit the page table setup in `kern/entry.S` and `kern/entrypgdir.c`. Immediately after we turn on paging, EIP is still a low number (a little over 1MB). At what point do we transition to running at an EIP above KERNBASE? What makes it possible for us to continue executing at a low EIP between when we enable paging and when we begin running at an EIP above KERNBASE? Why is this transition necessary?

1、让我们先回顾以下在`mem_init()`中，内核做了什么，首先在页目录中，插入了自身页目录的映射，然后映射了pages数组，内核栈和`KERNBASE`以上256MB的内存。

<img src=".\figure\6A42BD1C118C68DB57D84AD5F259E99A.png" alt="img" style="zoom:50%;" />

2、PTE_U没有被置为1。  

3、UPAGES的大小为PTSIZE = 1024 * 4KB = 4MB， 而一个struct PageInfo大小为8B， 所以一共有 4MB / 8B = 512K 页，而一页物理页的大小是4KB，所以最大支持的物理内存一共是512K * 4K = 2048MB = 2GB。  

4、页目录kern_pgdir + pages数组 + 页表。页目录占4KB, pages数组占4MB, 一共有512K页，那就是512K条页首地址，一条地址占32位，也就是4B，那么一共需要占512K * 4 = 2MB。所以一共占6MB + 4KB的开销。

5、暂时还不理解，可能以后会清楚。  



