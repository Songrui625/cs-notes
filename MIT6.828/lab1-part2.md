这个部分的主要内容是要了解内核是如何被引导的，大体过程如下：  

1. CPU先运行BIOS，控制权交到了BIOS。
2. BIOS将引导扇区装载进内存，控制权交给引导加载器。
3. 引导加载器将内核装载进内存，控制权交给内核。

引导加载器的两个作用：  

1、将CPU从实模式转换到32位的保护模式，因为只有在保护模式下软件才可以访问在CPU物理地址空间所有超过1MB的内存。  

2、通过x86特殊的I/O指令用直接访问IDE磁盘设备寄存器从硬盘读取内核。  

<!--more-->

### 引导扇区

PC机的软盘和硬盘被划分为**512字节**区域，称为**扇区**，扇区是磁盘的最小传输粒度（单元）：每次读写操作必须是一个或者多个扇区的大小并且要对齐扇区边界。如果磁盘是可引导的，那么第一个扇区就成为**引导扇区**，因为这里是boot loader（引导加载器）代码存放的地方。当BIOS找到可引导的软盘或硬盘，它会装载512字节的引导扇区进内存地址在0x7c00到0x7dff处，然后使用*jmp*指令设置cs：ip为0000:7c00,转移控制权给boot loader，这些地址是相当随意的，但它们对于PC是固定的和标准化的。  

现代PC机器已经不再是引导512字节的扇区了而是2048字节，而且BIOS可以从磁盘中加载更大的启动镜像到内存（不止一个扇区），然后把控制权交给它。（权当了解，与内容无太大关系）  

obj/boot/boot.asm，这是一个由makefile在编译之后创建的反汇编文件，它可以帮助我们轻松的所有引导程序代码在物理内存中的确切位置，而且可以更轻松地跟踪在GDB中逐步引导加载程序发生的情况。同样，obj/kern/kernel.asm包含一个JOS的反汇编文件，也对调试十分有帮助。  

xv6讲义推荐我们在0x7c00设置一个断点，通过 *b* **0x7c00*指令单步调试一趟，更深入的理解boot loader。

### Problem1

**At what point does the processor start executing 32-bit code? What exactly causes the switch from 16- to 32-bit mode?**

```bash
The target architecture is assumed to be i8086
[f000:fff0]    0xffff0: ljmp   $0xf000,$0xe05b
0x0000fff0 in ?? ()
+ symbol-file obj/kern/kernel
(gdb) b * 0x7c00
Breakpoint 1 at 0x7c00
(gdb) c
Continuing.
[   0:7c00] => 0x7c00:  cli    

Breakpoint 1, 0x00007c00 in ?? ()
(gdb) si
[   0:7c01] => 0x7c01:  cld    
0x00007c01 in ?? ()

......

(gdb) si
[   0:7c2d] => 0x7c2d:  ljmp   $0x8,$0x7c32
0x00007c2d in ?? ()
(gdb) si
The target architecture is assumed to be i386
=> 0x7c32:      mov    $0x10,%ax
0x00007c32 in ?? ()
```

通过 *c*指令和*si*指令进行单步调试，显示结果如上。  

重点关注这一行

```bash
The target architecture is assumed to be i386
=> 0x7c32:      mov    $0x10,%ax
```

这代表我们开始执行32位代码，i386是32位微处理器的统称，这一条显示的上一条指令是

```bash
[   0:7c2d] => 0x7c2d:  ljmp   $0x8,$0x7c32
```

就是这条*jmp*指令，让我们从16位模式切换到了32位模式，这也回答了Exercise 3的problem 1。  

### Problem 2

**What is the *last* instruction of the boot loader executed, and what is the *first* instruction of the kernel it just loaded?**

```bash
//boot/boot.S

protcseg:
  # Set up the protected-mode data segment registers
  movw    $PROT_MODE_DSEG, %ax    # Our data segment selector
  movw    %ax, %ds                # -> DS: Data Segment
  movw    %ax, %es                # -> ES: Extra Segment
  movw    %ax, %fs                # -> FS
  movw    %ax, %gs                # -> GS
  movw    %ax, %ss                # -> SS: Stack Segment
  
  # Set up the stack pointer and call into C.
  movl    $start, %esp
  call bootmain

  # If bootmain returns (it shouldn't), loop.
```

可以看到，boot.S最终调用了bootmain子程序，我们查看boot/main.c的源代码中的最后一条语句。

```c
// call the entry point from the ELF header
	// note: does not return!
	((void (*)(void)) (ELFHDR->e_entry))();

```

我们在obj/boot/boot.asm中找到这条语句的地址。

```c
	((void (*)(void)) (ELFHDR->e_entry))();
    7d81:	ff 15 18 00 01 00    	call   *0x10018
```

在这里设置断点，这就是boot loader的最后一条执行的指令。

```bash
(gdb) b *0x7d81
Breakpoint 4 at 0x7d81
(gdb) c
Continuing

=> 0x7d81:      call   *0x10018

Breakpoint 4, 0x00007d81 in ?? ()
(gdb) si
=> 0x10000c:    movw   $0x1234,0x472
0x0010000c in ?? ()
```

最后一条boot loader执行的指令就是

```bash
call   *0x10018
```

此时会从0x100018地址处取出跳转地址，跳转到该地址，也就是0x10000c，执行刚装载kernel第一条指令:

```bash
movw   $0x1234,0x472
```

### Problem3

**Where* is the first instruction of the kernel?**

```bash
=> 0x10000c:    movw   $0x1234,0x472
```

这是装载kernel后的第一条指令，我们去obj/boot/kernel.asm查找这条指令

```c
//obj/boot/kernel.asm

f010000c <entry>:
f010000c:	66 c7 05 72 04 00 00 	movw   $0x1234,0x472
f0100013:	34 12 
```

所以第一条指令在kern/entry.S中。

### problem4

**How does the boot loader decide how many sectors it must read in order to fetch the entire kernel from disk? Where does it find this information?**

要想回答问题，我们首先得对链接器、加载器和ELF（Executable and Linkable Format：可执行可链接格式）文件有一些了解，笔者强烈推荐《深入理解计算机系统》第七章链接作为参考资料。  

在类Unix系统下，假设我们有两个个C源文件，一个是main.c,另一个是sum.c，编译器把main.c翻译成一个*可重定位目标文件*main.o(事实上中间过程还有其他步骤)，sum.c也是如此，变成sum.o。最后，运行*链接器*，将main.o和sum.o以及一些必要的系统目标文件组合起来，形成了一个*可执行目标文件(executable object file)* prog。最后在linux shell上通过命令行输入它的名字运行程序。

```bash
./prog
```

shell会调用一个加载器的函数，它将可执行目标文件prog中的代码和数据复制到内存，将控制转移到这个程序的开头。  

以上提到的可重定位目标文件和可执行目标文件都是一个ELF格式的文件。我们lab下obj/kern/kernel就是一个可执行目标文件，说明他是一个ELF格式的文件。接下来看一下ELF文件的具体格式。

首先在最顶端的是一个ELF头（ELF Header）,接着是一个可变长度程序头（Program Header）列出了每个将要被加载到内存的程序节（我更喜欢叫程序段）。C语言在inc/elf.h头文件中定义了这些结构。而我们感兴趣的程序节主要是以下几种：  

- .text:程序的可执行指令，也就是已编译程序的机器代码。
- .rodata:只读数据，比如Ascii码表示的字符串字面量，printf语句中的格式串。
- .data:数据节保存的程序初始化诗句，帮你如已经初始化的全局和静态c变量，比如int x = 5。
- .bss:未初始化的全局和静态C变量，以及所有被初始化为0的全局或静态C变量。比如图 int x。值得注意的是，这个节只是个占位符，在目标文件中不占据实际空间。

最后是一个节头部表，不同程序节的位置和大小是由节头部表描述的，也就是说我们只要知道ELF头的大小初始地址、每个程序节的地址以及大小和节头部表的偏移地址以及大小，那么我们就可以算出整个程序的大小了。kernel的大小也就可以通过这种方式计算出来，那么也就回答了问题4。  

我们可以通过objdump -h 命令查看一个ELF文件的ELF格式。  

```bash
$ objdump -h obj/kern/kernel 

kernel:     file format elf32-i386

Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         00001acd  f0100000  00100000  00001000  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .rodata       000006bc  f0101ae0  00101ae0  00002ae0  2**5
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  2 .stab         00004291  f010219c  0010219c  0000319c  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  3 .stabstr      0000197f  f010642d  0010642d  0000742d  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .data         00009300  f0108000  00108000  00009000  2**12
                  CONTENTS, ALLOC, LOAD, DATA
  5 .got          00000008  f0111300  00111300  00012300  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  6 .got.plt      0000000c  f0111308  00111308  00012308  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  7 .data.rel.local 00001000  f0112000  00112000  00013000  2**12
                  CONTENTS, ALLOC, LOAD, DATA
  8 .data.rel.ro.local 00000044  f0113000  00113000  00014000  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  9 .bss          00000648  f0113060  00113060  00014060  2**5
                  CONTENTS, ALLOC, LOAD, DATA
 10 .comment      00000024  00000000  00000000  000146a8  2**0
                  CONTENTS, READONLY
```

我们只关注以上提到的几个节，其他节主要是用来保存调试信息的，它们不会被加载器加载到内存中。  

请特别注意.text节的 VMA(virtual memory address：虚拟内存地址)或者叫链接地址和 LMA(load memory address：加载内存地址)或者叫加载地址。节的加载地址是加载到内存中的内存地址。  

比如：.text节中，LMA = 00100000,那我们就会把这个节装载到内存中的00100000物理地址中，大小是0x1acd，那么末尾地址就是 0x100000 + 0x1acd = 0x101acd。  

而VMA = f01000000, 这就是反汇编文件kernel.asm中的第一行代码，为什么要有虚拟地址？这是为了进程的抽象：地址空间，让每个进程认为好像它认为拥有自己的私有物理内存的假象，其实操作系统（实际上是MMU：内存管理单元）完成了所有的工作，让不同的进程复用内存。  

也就是说，这个虚拟地址VMA其实是程序自己看到的被装载进内存中的地址，而实际的物理地址LMA也就是加载地址是被MMU进行地址转换得到的。  

我们再来查看一下boot.out的ELF格式：

```bash
$ objdump -h obj/boot/boot.out         

boot.out:     file format elf32-i386

Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         0000019c  00007c00  00007c00  00000074  2**2
                  CONTENTS, ALLOC, LOAD, CODE
  1 .eh_frame     0000009c  00007d9c  00007d9c  00000210  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  2 .stab         00000870  00000000  00000000  000002ac  2**2
                  CONTENTS, READONLY, DEBUGGING
  3 .stabstr      00000940  00000000  00000000  00000b1c  2**0
                  CONTENTS, READONLY, DEBUGGING
  4 .comment      0000002a  00000000  00000000  0000145c  2**0
                  CONTENTS, READONLY
```

可以看到，.text节的VMA 和 LMA是相同的，这样硬件就不用进行额外的地址转换。  

引导加载器使用ELF程序头决定了如何加载这些节到内存中。程序头指定了要加载到内存中的ELF对象的哪些部分，以及每个对象地址占用的目的地址，可以使用*objdump -x*指令查看程序头。  

```bash
$ objdump -x obj/kern/kernel

obj/kern/kernel:     file format elf32-i386
obj/kern/kernel
architecture: i386, flags 0x00000112:
EXEC_P, HAS_SYMS, D_PAGED
start address 0x0010000c

Program Header:
    LOAD off    0x00001000 vaddr 0xf0100000 paddr 0x00100000 align 2**12
         filesz 0x00007dac memsz 0x00007dac flags r-x
    LOAD off    0x00009000 vaddr 0xf0108000 paddr 0x00108000 align 2**12
         filesz 0x0000b6a8 memsz 0x0000b6a8 flags rw-
   STACK off    0x00000000 vaddr 0x00000000 paddr 0x00000000 align 2**4
         filesz 0x00000000 memsz 0x00000000 flags rwx

Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         00001acd  f0100000  00100000  00001000  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .rodata       000006bc  f0101ae0  00101ae0  00002ae0  2**5
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  2 .stab         00004291  f010219c  0010219c  0000319c  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  3 .stabstr      0000197f  f010642d  0010642d  0000742d  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .data         00009300  f0108000  00108000  00009000  2**12
                  CONTENTS, ALLOC, LOAD, DATA
  5 .got          00000008  f0111300  00111300  00012300  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  6 .got.plt      0000000c  f0111308  00111308  00012308  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  7 .data.rel.local 00001000  f0112000  00112000  00013000  2**12
                  CONTENTS, ALLOC, LOAD, DATA
  8 .data.rel.ro.local 00000044  f0113000  00113000  00014000  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  9 .bss          00000648  f0113060  00113060  00014060  2**5
                  CONTENTS, ALLOC, LOAD, DATA
 10 .comment      00000024  00000000  00000000  000146a8  2**0
                  CONTENTS, READONLY
...
```

ELF对象需要加载到内存的区域是标记为LOAD的区域，程序头信息给出了虚拟地址信息，物理地址信息以及加载区域的大小（文件大小和内存大小）。  

回到boot/main.c,  ph-p_pa 每个程序头的包含了该段的目标物理地址。   

我来描述一下main.c的任务，读取ELF格式文件 obj/boot/kernel，把每个节和程序头的物理地址都加载到内存中。看下源代码：

```c
#define ELFHDR		((struct Elf *) 0x10000) // scratch space 暂存空间
```

这就是ELF头所在的地址，把他转化为一个指向ELF结构体的指针。

```c
ph = (struct Proghdr *) ((uint8_t *) ELFHDR + ELFHDR->e_phoff);
```

将ELF头地址加上程序头入口地址就能得到程序头地址，这里利用强制转换成了一个指向程序头结构体的指针。

```c
eph = ph + ELFHDR->e_phnum;
```

这里e_phnum就是程序头的数量，由于每一个程序头都记录了要装载到内存的程序的信息，比如程序所在的地址，要加载到内存的物理地址等等，eph就是最后一个的程序头的地址，这样我们就可以遍历每个程序头，将它们的代码读取到内存的指定位置中。

```c
	for (; ph < eph; ph++)
		// p_pa is the load address of this segment (as well
		// as the physical address)
		readseg(ph->p_pa, ph->p_memsz, ph->p_offset);
```

其实就是从kernel中的每个程序头的offset起始地址(ph_offset)开始读取程序段到程序头要加载到内存中的物理地址(ph->p_pa),大小是ph->p_memsz个字节，而每次读取程序段过程中又会去读扇区。所以problem4的问题就可以回答了，程序头有多少个条目，就会读取多少次程序段，读程序段的次数乘以每次读扇区的数目也就是读内核所需要的扇区数目。

内核到此加载完毕。





