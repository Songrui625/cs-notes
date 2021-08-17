---
title: 'MIT6.828 HW:xv6 locking'
date: 2021-03-20 15:02:11
tags: xv6 
---

在这个作业中将会探究在中断和锁之间的交互。<!--more-->

## Don't do this

确保你理解如果xv6内核运行了下面的代码段会发生什么？

```c
  struct spinlock lk;
  initlock(&lk, "test lock");
  acquire(&lk);
  acquire(&lk);
```

在`spinlock.c:acquire()`中

```c
if(holding(lk))
    panic("acquire");
```

如果已经持有锁了，那将会panic。  

## Interrupts in ide.c

`acquire`会使用`cli`指令(通过`pushcli()`)确保关闭本地处理器上的中断，并且中断一直保持关闭直到释放该处理器所持有的最后一个锁。（这时候会使用sti开启中断）。  

我们来看一下如果我们在持有`ide`锁时打开了中断会发生什么？在`acquire()`调用后添加一个`sti()`调用，以及在`release()`调用前添加一个`cli()`。重新build内核然后引导，内核可能会看会出现恐慌。  

```c
void
iderw(struct buf *b)
{
  ...
  acquire(&idelock);  //DOC:acquire-lock
  sti();
  ...
  cli();
  release(&idelock);
}
```



```bash
Booting from Hard Disk..xv6...
cpu1: starting 1
cpu0: starting 0
sb: size 1000 nblocks 941 ninodes 200 nlog 30 logstart 2 inodestart 32 bmap start 58
lapicid 1: panic: sched locks
 80103d95 80103f16 80105bcf 8010593c 80100191 80101ada 80100ad2 801055d8 80104a9d 80105bf5
```

开中断导致重复acquire？暂时还没理解。

## Interrupts in file.c

去掉之前的代码。现在我们来看看如果在持有`file_table_lock`的同时打开中断会发生什么？锁会保护文件描述符表，应用程序打开或者关闭文件时内核会修改表中的内容。同上面一样操作，相关文件在`file.c:filealloc()`中。

```c
// Allocate a file structure.
struct file*
filealloc(void)
{
  struct file *f;

  acquire(&ftable.lock);
+ sti();
  for(f = ftable.file; f < ftable.file + NFILE; f++){
    if(f->ref == 0){
      f->ref = 1;
+     cli();
      release(&ftable.lock);
      return f;
    }
  }
+ cli();
  release(&ftable.lock);
  return 0;
}
```

但是，我仍然出现了panic：

```bash
Booting from Hard Disk..xv6...
cpu1: starting 1
cpu0: starting 0
sb: size 1000 nblocks 941 ninodes 200 nlog 30 logstart 2 inodestart 32 bmap start 58
lapicid 0: panic: sched locks
 80103da5 80103f26 80105bdf 8010594c 80105273 80104aad 80105c05 8010594c 0 0
```

按照教授说的，我可以去买彩票了。  

## xv6 lock implementation

对共享资源的访问可能会导致重复设置，所以需要在持有锁的时候清理pcs。

