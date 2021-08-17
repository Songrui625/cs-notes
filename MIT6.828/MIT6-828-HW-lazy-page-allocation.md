---
title: 'MIT6.828 HW:lazy page allocation'
date: 2021-03-18 11:40:44
tags: xv6
---


操作系统可以使用页表硬件进行许多巧妙技巧之一就是堆内存的延迟分配。<!--more-->Xv6 应用 通过 `sbrk()`系统调用向内核申请堆内存。`sbrk()`分配物理内存然后映射到进程的虚拟地址空间。有程序分配内存但从不使用，例如大型稀疏数组。内核会延迟分配每个内存页面，直到应用程序尝试使用该页面为止——如页面错误所示。在这个lab中我们得实现延迟分配特性。  

## Part One: Eliminate allocation from sbrk()

第一个任务就是从`sbrk()`系统调用实现中删除页面分配, 这个函数是`sysproc.c:sys_sbrk()`。`sbrk(k)`系统调用会为进程的内存大小增加n个字节，然后返回新的分配地址。(ie., old size)。我们新的`sbrk(n)`应该只是增加进程n字节的size(myproc()->sz)，然后返回原来的old size。它不应该分配内存——所以我们应该删除对`growproc()`的调用(**但是你仍然需要增加进程的size**)。  

```c
int
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
+ // if(growproc(n) < 0)
+ //   return -1;
+ myproc()->sz += n; //尽管不分配内存，但是我们仍然需要增加进程的size。
  return addr;
}
```

想一下修改后会导致什么？什么会被break掉。  

在shell中输入echo hi

```bash
$ echo hi
pid 3 sh: trap 14 err 6 on cpu 1 eip 0x11c8 addr 0x4004--kill proc
```

打印出来的信息是由内核的trap hanlder产生的。它已经捕获到了一个页面错误(trap 14, or T_PGFLT), xv6内核不知道如何处理。确保你理解为什么会发生页面错误。`"addr 0x4004"`表示造成页面错误的虚拟地址。  

发生页面错误的过程如下：sh在解析fork一个子进程，会去解析cmd，解析过程中会调用`execcmd()`，此时会调用`malloc`函数来获取内存，而`malloc()`中调用了`morecore()`, `morecore()`调用了`sbrk()`, 然后会trap进内核，调用`sys_sbrk()`系统调用，我们也知道我们前面没有分配堆内存，而是直接返回了进程的old size。后面的具体哪条代码出错我也不得而知。

```c
//PAGEBREAK!
// Constructors

struct cmd*
execcmd(void)
{
  struct execcmd *cmd;

  cmd = malloc(sizeof(*cmd)); //这里出现错误
  memset(cmd, 0, sizeof(*cmd));
  cmd->type = EXEC;
  return (struct cmd*)cmd;
}
```

## Part Two: Lazy allocation

修改`trap.c`中的代码，通过映射一个新分配的物理页地址到错误地址来响应用户空间导致的页面错误，然后返回用户空间让进程继续执行。我们需要在`cprintf`调用之前添加自己的代码。代码没有必要覆盖所有的corner case和error situations，只需要让sh能执行简单的echo 和 ls这样的命令就足够了。    

看完所有的hint，就知道如何写了。参考`vm.c:allocuvm()`写法。

```c
  default:
    if(myproc() == 0 || (tf->cs&3) == 0){
      // In kernel, it must be our mistake.
      cprintf("unexpected trap %d from cpu %d eip %x (cr2=0x%x)\n",
              tf->trapno, cpuid(), tf->eip, rcr2());
      panic("trap");
    }
   +extern int mappages(pde_t *pgdir, void *va, uint size, uint pa, int perm);
   +if(tf->trapno == T_PGFLT) {
   +  char *mem;
   +  uint a;
   +  mem = kalloc();
   +  memset(mem, 0, PGSIZE);
   +  if (mappages(myproc()->pgdir, (char*) a, PGSIZE, V2P(mem), PTE_W|PTE_U) == 0)
   +    return;
   +}
```

optional challenge:处理`sbrk()`参数为负数的情况。就做个if判断就好了。但是如果参数过大，以及一些错误情况还不知道如何处理。

```c
int
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;
  // cprintf("n:%p\n", n);
  addr = myproc()->sz;
  // if(growproc(n) < 0)
  //   return -1;
  if (n >= 0)
    myproc()->sz += n;
  if (n < 0)
    myproc()->sz -= n;
  return addr;
}
```

