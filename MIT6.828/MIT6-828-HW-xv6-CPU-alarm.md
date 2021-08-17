---
title: 'MIT6.828 HW:xv6 CPU alarm'
date: 2021-03-19 16:21:11
tags: xv6
---

这个作业我们要为xv6添加一个特性，定期的警告进程它使用了多少CPU时间。<!--more-->

我们应该添加一个新的系统调用`alarm(interval, handler)`。如果一个应用进程调用了`alarm(n, fn)`，那么CPU每消耗n "ticks"，内核就会导致应用进程调用fn。当fn返回时，应用进程会从上次的地方恢复。一个tick就是xv6中十分随机的时间单位，它由硬件计时器多久产生一次产生中断决定。也就是硬件计时器每产生一次中断，那么就是一次tick。  

添加一个system call在之前的作业已经实现过，不再过多赘述，只阐述需要注意的几点。  

如果实现完后没有看到应该看到的显示，那么可以增加迭代次数为原来的十倍。

```c
#include "types.h"
#include "stat.h"
#include "user.h"

void periodic();

int
main(int argc, char *argv[])
{
  int i;
  printf(1, "alarmtest starting\n");
  alarm(10, periodic);
//for(i = 0; i < 25*50000; i++){
+ for(i = 0; i < 25*500000; i++){
    if((i % 250000) == 0)
      write(2, ".", 1);
  }
  exit();
}

void
periodic()
{
  printf(1, "alarm!\n");
}
```

我们知道每产生一次interrupt或者trap，都会陷入内核，然后调用`trap.c:trap(struct trapframe *tf)`。

```c
  switch(tf->trapno){
  case T_IRQ0 + IRQ_TIMER:
    if(cpuid() == 0){
      acquire(&tickslock);
      ticks++;
      wakeup(&ticks);
      release(&tickslock);
    }
    if(myproc() && (tf->cs & 3) == 3){ // myproc() != 0表示进程正在运行(也就是正在消耗CPU)， tf->cs & 3 表示中断由用户空间产生。
        myproc()->alarmcount++;        // count++直到alarmticks
        if(myproc()->alarmticks == myproc()->alarmcount){  // 到达了最大规定的最大tick数，此时应该调用handler函数。
            myproc()->alarmcount = 0;//置0以便下一次允许。
          tf->esp -= 4;    //开辟一个单元的栈空间
          *((uint *)(tf->esp)) = tf->eip; //将旧的eip压栈,这样handler函数调用结束后就会从栈顶pop到eip中，执行下一条指令。
          // eip指向新的函数地址，这样中断返回之后就会去执行handler函数，
          tf->eip =(uint) myproc()->alarmhandler;
        }
      }
    lapiceoi();
    break;
```

注意C语言的calling convention，会把旧的eip(函数返回地址)压栈，然后eip指向call address，调用最后执行ret指令pop 到 eip中，eip指向函数返回地址继续执行。而在中断中，我们需要自己去完成将旧的eip压栈。