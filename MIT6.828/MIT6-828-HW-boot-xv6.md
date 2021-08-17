---
title: 'MIT6.828 HW:boot xv6'
date: 2021-01-07 17:41:29
tags: xv6

---

本章作业的内容是引导xv6以及理解堆栈如何进行初始化。

<!--more-->

## Finding and breaking at an address

找到内核的入口点，_start标号的地址。

```bash
$ nm kernel | grep _start
8010a48c D _binary_entryother_start
8010a460 D _binary_initcode_start
0010000c T _start
```

## Exercise: What is on the stack?

```bash
(gdb) info reg
eax            0x0                 0
ecx            0x0                 0
edx            0x1f0               496
ebx            0x10094             65684
esp            0x7bdc              0x7bdc
ebp            0x7bf8              0x7bf8
esi            0x10094             65684
edi            0x0                 0
eip            0x10000c            0x10000c
eflags         0x46                [ IOPL=0 ZF PF ]
cs             0x8                 8
ss             0x10                16
ds             0x10                16
es             0x10                16
fs             0x0                 0
gs             0x0                 0
fs_base        0x0                 0
gs_base        0x0                 0
k_gs_base      0x0                 0
cr0            0x11                [ ET PE ]
cr2            0x0                 0
cr3            0x0                 [ PDBR=0 PCID=0 ]
--Type <RET> for more, q to quit, c to continue without paging--
```

我们需要额外关注和堆栈相关的寄存器: ebp 和 esp。

```bash
(gdb) x/24x $esp
0x7bdc: 0x00007d97      0x00000000      0x00000000      0x00000000
0x7bec: 0x00000000      0x00000000      0x00000000      0x00000000
0x7bfc: 0x00007c4d      0x8ec031fa      0x8ec08ed8      0xa864e4d0
0x7c0c: 0xb0fa7502      0xe464e6d1      0x7502a864      0xe6dfb0fa
0x7c1c: 0x16010f60      0x200f7c78      0xc88366c0      0xc0220f01
0x7c2c: 0x087c31ea      0x10b86600      0x8ed88e00      0x66d08ec0
```

现在我们并不知道这些非0值代表什么，我们可以通过qemu知道此时虚拟机正在引导内核，所以我们查看引导相关的文件，搜索`_start`，在entry.S中可以看到:

```assembly

# By convention, the _start symbol specifies the ELF entry point.
# Since we haven't set up virtual memory yet, our entry point is
# the physical address of 'entry'.
.globl _start
_start = V2P_WO(entry)

# Entering xv6 on boot processor, with paging off.
.globl entry
entry:
  # Turn on page size extension for 4Mbyte pages
  movl    %cr4, %eax
  orl     $(CR4_PSE), %eax
  movl    %eax, %cr4
  # Set page directory
  movl    $(V2P_WO(entrypgdir)), %eax
  movl    %eax, %cr3
  # Turn on paging.
  movl    %cr0, %eax
  orl     $(CR0_PG|CR0_WP), %eax
  movl    %eax, %cr0
```

注释提到，因为还没有建立虚拟内存，所以`_start`符号代表ELF文件的入口点，这里使用了一个`V2P_WO`宏来实现entry虚拟地址向物理地址转换。所以说明这条指令的物理地址在0x10000c， 而eip指向的也是此。而entry是一个过程(函数), 所以系统此时应该是调用了entry函数，回想x86调用函数机制，首先会把`旧的eip值(也就是函数调用的下一条指令的地址)`压栈，然后把`函数的首地址赋给eip`，如果有参数还会把参数从右往左压入栈中。我们在`bootblock.asm`中可以找到如下代码：

```assembly
  entry();
    7d91:	ff 15 18 00 01 00    	call   *0x10018
}
    7d97:	8d 65 f4             	lea    -0xc(%ebp),%esp
```

调用entry()函数的`下一条指令的地址正是7d97`，这就是栈顶的内容`0x00007d97`。而栈顶的下一个非零值`0x00007c4d`又是什么呢？我们按照之前的思路，`entry()`是在`bootmain.c/bootmain()`中被调用的，所以`0x00007c4d`可能就是`调用bootmain()的下一条指令的地址`, 我们去`bootblock.asm`中可以找到：

```assembly
  # Set up the stack pointer and call into C.
  movl    $start, %esp
    7c43:	bc 00 7c 00 00       	mov    $0x7c00,%esp
  call    bootmain
    7c48:	e8 fc 00 00 00       	call   7d49 <bootmain>

  # If bootmain returns (it shouldn't), trigger a Bochs
  # breakpoint if running under Bochs, then loop.
  movw    $0x8a00, %ax            # 0x8a00 -> port 0x8a00
    7c4d:	66 b8 00 8a          	mov    $0x8a00,%ax
```

`call bootmain`下一条指令的地址正是`0x00007d97`， 也可以说是bootmain函数的返回地址。中间那些0值实际上就是各个寄存器(ebp, edi, esi, ebx等)的值。  

试着回答一些问题：  

1、在`bootblock.asm`0x7c00处设一个断点，单步执行指令，`stack pointer`在`bootasm.S`中的哪里被初始化的？(单步执行指令知道你看到将一个值mov到%esp，其中esp是堆栈指针寄存器)  

```assembly
  # Set up the stack pointer and call into C.
  movl    $start, %esp
    7c43:	bc 00 7c 00 00       	mov    $0x7c00,%esp
```

```assembly
.code32  # Tell assembler to generate 32-bit code now.
start32:
  # Set up the protected-mode data segment registers
  movw    $(SEG_KDATA<<3), %ax    # Our data segment selector
  movw    %ax, %ds                # -> DS: Data Segment
  movw    %ax, %es                # -> ES: Extra Segment
  movw    %ax, %ss                # -> SS: Stack Segment
  movw    $0, %ax                 # Zero segments not ready for use
  movw    %ax, %fs                # -> FS
  movw    %ax, %gs                # -> GS

  # Set up the stack pointer and call into C.
  movl    $start, %esp
  call    bootmain
```

可以看到是在内核建立保护模式数据段寄存器的时候，对堆栈指针进行了初始化，然后调用C代码。  

2、单步执行了call bootmain后，此时堆栈上值是什么？  

```bash
(gdb) si
=> 0x7c48:      call   0x7d49
0x00007c48 in ?? ()

(gdb) si
=> 0x7d49:      endbr32 
0x00007d49 in ?? ()
(gdb) x/24x $esp
0x7bfc: 0x00007c4d      0x8ec031fa      0x8ec08ed8      0xa864e4d0
0x7c0c: 0xb0fa7502      0xe464e6d1      0x7502a864      0xe6dfb0fa
0x7c1c: 0x16010f60      0x200f7c78      0xc88366c0      0xc0220f01
0x7c2c: 0x087c31ea      0x10b86600      0x8ed88e00      0x66d08ec0
0x7c3c: 0x8e0000b8      0xbce88ee0      0x00007c00      0x0000fce8
0x7c4c: 0x00b86600      0xc289668a      0xb866ef66      0xef668ae0
```

此时堆栈栈顶值为`7c4d`， 正是`call bootmain`下一条指令的地址，也就是`call bootmain的返回地址`。  

3、bootmain的第一条汇编指令对堆栈做了什么？  

```bash
    7d4d:	55                   	push   %ebp
```

将`ebp寄存器`压入栈中。而此时`ebp寄存器`中值为`0`, 那么栈顶的值应该为0，下一个就是7c4d.

```bash
(gdb) x/24x $esp
0x7bf8: 0x00000000      0x00007c4d      0x8ec031fa      0x8ec08ed8
```

4、查找将eip改变为0x10000c的call指令，它对堆栈做了什么？  

```bash
(gdb) c
Continuing.
=> 0x7d91:      call   *0x10018

=> 0x10000c:    mov    %cr4,%eax
0x0010000c in ?? ()
(gdb) info reg
eax            0x0                 0
ecx            0x0                 0
edx            0x1f0               496
ebx            0x10094             65684
esp            0x7bdc              0x7bdc
ebp            0x7bf8              0x7bf8
esi            0x10094             65684
edi            0x0                 0
eip            0x10000c            0x10000c             // here
eflags         0x46                [ IOPL=0 ZF PF ]
cs             0x8                 8
ss             0x10                16
ds             0x10                16
es             0x10                16
fs             0x0                 0
```



现在我们已经知道，当调用call指令时，会把call指令的下一条指令(旧的eip)的地址压栈，然后改变eip的值。

```assembly
  entry();
    7d91:	ff 15 18 00 01 00    	call   *0x10018
}
    7d97:	8d 65 f4             	lea    -0xc(%ebp),%esp
```

所以`7d97`会被压栈，`esp寄存器`中的值为`0x7d97`