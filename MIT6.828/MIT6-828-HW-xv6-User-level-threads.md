---
title: 'MIT6.828 HW:xv6 User-level threads'
date: 2021-04-07 10:46:28
tags: xv6
---

在这次作业，我们需要通过实现线程之间的上下文切换代码完成一个简单的用户级线程包。<!--more-->

## Switching threads

`uthread`创建了两个线程并在他们之间来回切换。每个线程打印"my thread ..."然后yields来让另一个线程有机会运行。  

我们需要完成`thread_switch.S`。`uthread.c`拥有两个全局变量`current_thread`和`next_thread`。每个都是一个指向结构体`thread`的指针。`thread`结构体拥有一个线程的堆栈，和一个保存了的堆栈指针(sp, 指向线程的堆栈)。`uthread_switch`的任务是保存当前线程的状态到`current_thread`指向的结构中，恢复`next_thread`的状态，让`current_thread`指向`next_thread`指向的位置，这样当`uthread_switch`有返回时，`next_thread`正在运行，它就是当前的`current_thread`。  

研究下`thread_create`，它为一个新的线程建立了初始化堆栈。它为`thread_switch`应该做什么提供了提示。`thread_switch`的意图是使用汇编指令`popal`和`pushal`来恢复和保存八个x86寄存器。  

我们还需要了解`struct thread`的在C编译器下的内存布局: 

```c
    --------------------
    | 4 bytes for state|
    --------------------
    | stack size bytes |
    | for stack        |
    --------------------
    | 4 bytes for sp   |
    --------------------  <--- current_thread
         ......

         ......
    --------------------
    | 4 bytes for state|
    --------------------
    | stack size bytes |
    | for stack        |
    --------------------
    | 4 bytes for sp   |
    --------------------  <--- next_thread
```

为了写入`current_thread`指向的sp字段，可以编写如下汇编：

```assembly
movl current_thread, %eax
movl %esp, (%eax)
```

这样就会在`current_thread->sp`保存`%esp`，这样有效是因为`sp字段`在结构体中的偏移量是0。  

有了以上的知识铺垫，我们就可以开始实现`thread_switch`了。

```assembly
	.text

/* Switch from current_thread to next_thread. Make next_thread
 * the current_thread, and set next_thread to 0.
 * Use eax as a temporary register; it is caller saved.
 */
	.globl thread_switch
thread_switch:
	/* YOUR CODE HERE */
	// 保存当前线程的八个寄存器(包括程序计数器eip)。 
    pushal
    //将当前堆栈指针保存到current_thread->sp中
    movl current_thread, %eax
    movl %esp, (%eax)
    //将next_thread的地址保存到当前堆栈指针,此时就完成了
    //堆栈指针从指向current_thread到指向next_thread的转换
    movl next_thread, %eax
    movl (%eax), %esp
    
    popal
	//此时可以将next_thread的地址保存到current_thread中。
	//此时current_thread就指向了next_thread指向的位置。
    movl next_thread, %eax
    movl %eax, current_thread
    //此时%eax保存的是old current_thread的地址。
    movl $0, next_thread
    movl %eax, next_thread
	ret				/* pop return address from stack */
```

```bash
(gdb) p/x next_thread->sp
$4 = 0x4ae8
(gdb) x/9x next_thread->sp
0x4ae8 :      0x00000000      0x00000000      0x00000000      0x00000000
0x4af8 :      0x00000000      0x00000000      0x00000000      0x00000000
0x4b08 :      0x000000d8
```

位于`next_thread`堆栈顶部的`0xd8`是什么地址？  

是`uthread_switch()`的返回地址。