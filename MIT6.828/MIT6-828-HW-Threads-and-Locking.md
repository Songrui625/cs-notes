---
title: 'MIT6.828 HW:Threads and Locking'
date: 2021-03-20 11:30:06
tags: xv6 
---

在本次作业中，将会在一个hash table上使用线程和锁来探索并行编程（我觉得其实是并发）。<!--more-->  

编译给出的代码，分别用单线程和双线程运行程序，结果如下：

```c
// two thread
1: put time = 0.003338
0: put time = 0.003389
0: get time = 7.684335
0: 17480 keys missing
1: get time = 7.684335
1: 17480 keys missing
completion time = 7.687856
```

```c
// single thread
$ ./a.out 1
0: put time = 0.004073
0: get time = 6.929189
0: 0 keys missing
completion time = 6.933433
```

线程会运行在两个阶段：get 和 put。通过结果对比，我们可以看到总完成时间其实是差不多的，并不是说多线程就一定块。但是双线程在get阶段完成单线程两倍的工作量，换句话说，单线程做一份工作，而双线程是两份工，所以双线程实现了两倍的并行加速。  

但双线程也不是没有缺点，那就是在get阶段会丢失key，而单线程却不会丢失。  

```bash
static 
void put(int key, int value)
{
  int i = key % NBUCKET;
  insert(key, value, &table[i], table[i]);
}
```

我们可以看下这段代码，这是一个静态函数，也就是全局只有一个，table是一个共享变量，也就是说两个线程都可以访问，我们想象下如下场景：  

首先线程1被调度运行，key % NBUCKET的值计算完后存储到了 eax 寄存器中，还没有写到i中，这时发生了时钟中断，线程2被调度运行，此时变量i计算完毕，被插入到hash table中，此时又发生了时钟中断，线程1被调度运行，会把eax寄存器的值写到i的内存单元中，并且插入到hash table中，导致重复操作，所以hashtable中的某些项是空的。  

为了保证线程之间不会产生race condition，我们需要对操作进行上锁。我们知道某些key确实是由于多线程的put操作造成的，我们先选择在put操作加锁。

```c
static 
void put(int key, int value)
{
  pthread_mutex_lock(&lock);
  int i = key % NBUCKET;
  insert(key, value, &table[i], table[i]);
  pthread_mutex_unlock(&lock);
}
```

这时我们可以保证双线程的get操作已经不会确实key了，因为每个put都被正确执行。

```bash
$ ./a.out 2               
0: put time = 0.022687
1: put time = 0.022984
1: get time = 6.194159
1: 0 keys missing
0: get time = 6.195507
0: 0 keys missing
completion time = 6.218766
```

但是我们能保证每个get操作都是正确的吗？暂时我也不清楚。