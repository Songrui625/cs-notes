在这个部分，我们更详细的研究JOS内核（最后还会写一些代码）。与引导加载器一样，内核从一些汇编语言代码开始，这些代码会进行设置，以便C语言代码可以正确执行。

<!--more-->

### 使用虚拟内存来解决位置依赖问题

如上一章看到的，引导加载器的链接地址和加载地址完全匹配，但是内核的链接地址和加载地址之间存在相当大的差异。我在这里贴一下上一章的代码。

```bash
$ objdump -h obj/boot/boot.out

obj/boot/boot.out:     file format elf32-i386

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

```bash
$ objdump -h obj/kern/kernel

obj/kern/kernel:     file format elf32-i386

Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         00001acd  f0100000  00100000  00001000  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
```

VMA和LMA差了f0000000。  

操作系统内核通常会被链接到很高的虚拟地址运行，比如0xf0100000,这是为了将处理器虚拟地址空间的低部分留给用户程序使用。  

很多机器物理地址范围都无法达到0xf0100000,所以我们不能指望把内核存储在那里。相反，我们可以使用处理器的内存管理硬件将虚拟地址0xf0100000(内核代码期望的运行地址)映射到物理地址0x0100000(引导加载器将内核加载到物理内存中的地址)。通过这种方法，尽管内核的虚拟地址很高但是留下了足够多的地址空间以供用户进程使用，它会被加载到PC RAM中物理内存中1MB的位置，就在BIOS ROM的上方。这种方法要求PC具有几M字节的物理内存，这样才能使物理地址0x00100000起作用。  

现在，我们只需要映射物理内存的前4MB字节，就足够让内核启动并运行。我们使用kern/entrypgdir.c中的手写的，静态初始化的页目录和页表来完成这个工作。现在没有必要了解具体细节，知道实现方式就可以了。  

在`kern/entry.S`代码中设置`CR0_PG`标志之前，内存引用被看作是物理地址（严格地说，它们是线性地址，但是`boot/boot.S`设置了一个标识符来从线性地址映射到物理地址，然后永远不会改变标识符）。一旦`CR0_PG`标志被设置，内存引用会被当作是虚拟地址，虚拟内存硬件会将这些引用转换为物理地址。  

**Exercise7**

> Use QEMU and GDB to trace into the JOS kernel and stop at the `movl %eax, %cr0`. Examine memory at 0x00100000 and at 0xf0100000. Now, single step over that instruction using the stepi GDB command. Again, examine memory at 0x00100000 and at 0xf0100000. Make sure you understand what just happened.
>
> What is the first instruction *after* the new mapping is established that would fail to work properly if the mapping weren't in place? Comment out the `movl %eax, %cr0` in `kern/entry.S`, trace into it, and see if you were right.

```bash
=> 0x100025:    mov    %eax,%cr0
0x00100025 in ?? ()

//执行前
(gdb) x/8x 0x100000
0x100000:       0x1badb002      0x00000000      0xe4524ffe      0x7205c766
0x100010:       0x34000004      0x2000b812      0x220f0011      0xc0200fd8
(gdb) x/8x 0xf0100000
0xf0100000 <_start-268435468>:  0x00000000      0x00000000      0x00000000      0x00000000
0xf0100010 <entry+4>:   0x00000000      0x00000000      0x00000000      0x0000000

(gdb) si
=> 0x100028:    mov    $0xf010002f,%eax
0x00100028 in ?? ()
//执行后
(gdb) x/8x 0x100000
0x100000:       0x1badb002      0x00000000      0xe4524ffe      0x7205c766
0x100010:       0x34000004      0x2000b812      0x220f0011      0xc0200fd8

(gdb) x/8x 0xf0100000
0xf0100000 <_start-268435468>:  0x1badb002      0x00000000      0xe4524ffe      0x7205c766
0xf0100010 <entry+4>:   0x34000004      0x2000b812      0x220f0011      0xc0200fd8
```

两者唯一的区别就是执行了movl %eax, %cr0指令后，虚拟地址0xf0100000被映射到了实际物理地址0x00100000处。

现在我们把movl %eax, %cr0注释掉，重新跟踪。

```bash
=> 0x10001d:    mov    %cr0,%eax
0x0010001d in ?? ()
(gdb) si
=> 0x100020:    or     $0x80010001,%eax
0x00100020 in ?? ()
(gdb) si
=> 0x100025:    mov    $0xf010002c,%eax
0x00100025 in ?? ()
(gdb) x/8x 0x100000
0x100000:       0x1badb002      0x00000000      0xe4524ffe      0x7205c766
0x100010:       0x34000004      0x2000b812      0x220f0011      0xc0200fd8
(gdb) x/8x 0xf0100000
0xf0100000 <_start-268435468>:  0x00000000      0x00000000      0x00000000      0x00000000
0xf0100010 <entry+4>:   0x00000000      0x00000000      0x00000000      0x00000000
```

可以看到，注释掉`movl %eax, %cr0`后，虚拟地址0xf0100000就无法映射到内存地址0x100000了。这也印证了我们上面的描述，一旦设置了`CR0_PG`标志后，内存引用会被当作是虚拟地址。

### 格式化打印到控制台

很多人把`printf()`函数认为是理所当然的，有时甚至认为它是C语言的“原语”。但是，在一个操作系统内核中，我们必须亲自实现所有I/O操作。  

阅读`kern/printf.c`, `lib/printfmt.c`和`kern/console.c`.确保理解它们之间的关系。在后面的lab将会理解为什么`printfmt.c`是位于独立的lib目录中。  

```c
//kern/printf.c
// Simple implementation of cprintf console output for the kernel,
// based on printfmt() and the kernel console's cputchar().

#include <inc/types.h>
#include <inc/stdio.h>
#include <inc/stdarg.h>


static void
putch(int ch, int *cnt)
{
	cputchar(ch);
	*cnt++;
}

int
vcprintf(const char *fmt, va_list ap)
{
	int cnt = 0;

	vprintfmt((void*)putch, &cnt, fmt, ap);
	return cnt;
}
 
```

我们通过函数调用关系可以知道，`console.c`中的`cputchar()`被封装成了`putch（）`，向控制台输出一个字符,而`printfmt.c`中的`vprintfmt（）`被封装成了`vcprintf()`，作用是打印一个字符串，`vcprintf()`又被封装成`cprinf()`供用户使用。  

**Excersize8**

> We have omitted a small fragment of code - the code necessary to print octal numbers using patterns of the form "%o". Find and fill in this code fragment.

```c
case 'o':
// Replace this with your code.

putch('0', putdat);
num = getuint(&ap, lflag);
base = 8;
goto number;
```

回答以下问题：

**1.Explain the interface between `printf.c` and `console.c`. Specifically, what function does `console.c` export? How is this function used by `printf.c`?**  

具体答案就在上面。

**2.Explain the following from `console.c`:**

```c
    //crt:阴极射线显像管，后文统称显示屏
	if (crt_pos >= CRT_SIZE) {
    	//如果显示屏显示字符的位置超过了显示屏最大显示字符字数
              int i;
              memmove(crt_buf, crt_buf + CRT_COLS, (CRT_SIZE - CRT_COLS) * sizeof(uint16_t));
        	  //将控制台已经存在的字符移到上一行。
              for (i = CRT_SIZE - CRT_COLS; i < CRT_SIZE; i++)
                      crt_buf[i] = 0x0700 | ' ';
        	  //然后擦除当前行的所有字符，也就是每个字符用空字符‘ ’代替
              crt_pos -= CRT_COLS;
              //更新当前字符位置
      }
```

**3.Trace the execution of the following code step-by-step:**

```c
int x = 1, y = 3, z = 4;
cprintf("x %d, y %x, z %d\n", x, y, z);
```

- In the call to `cprintf()`, to what does `fmt` point? To what does `ap` point?

在kern/init.c中的i386_init()中添加以上代码。

```c
void
i386_init(void)
{
	extern char edata[], end[];

	// Before doing anything else, complete the ELF loading process.
	// Clear the uninitialized global data (BSS) section of our program.
	// This ensures that all static/global variables start out zero.
	memset(edata, 0, end - edata);

	// Initialize the console.
	// Can't call cprintf until after we do this!
	cons_init();

	cprintf("6828 decimal is %o octal!\n", 6828);
	
+	int x = 1, y = 3, z = 4;
+Lab1_exercise8_3:
+	cprintf("x %d, y %x, z %d\n", x, y, z);

	// Test the stack backtrace function (lab 1 only)
	test_backtrace(5);

	// Drop into the kernel monitor.
	while (1)
		monitor(NULL);
}
```

然后运行，make qemu，在反汇编代码kern/kernel.asm中找到标号：Lab——exercise8_3。

```c
	int x = 1, y = 3, z = 4;
Lab1_exercise8_3:
	cprintf("x %d, y %x, z %d\n", x, y, z);
f01000f0:	6a 04                	push   $0x4
f01000f2:	6a 03                	push   $0x3
f01000f4:	6a 01                	push   $0x1
f01000f6:	8d 83 6a 08 ff ff    	lea    -0xf796(%ebx),%eax
f01000fc:	50                   	push   %eax
f01000fd:	e8 9b 09 00 00       	call   f0100a9d <cprintf>
```

可以看到参数是从右到左压入栈顶的，打上断点，进行debug，单步调试。

```bash
=> 0xf0100a9d <cprintf>:        endbr32 
cprintf (fmt=0xf0101b72 "x %d, y %x, z %d\n") at kern/printf.c:27
27      {
(gdb) x/s 0xf0101b72
0xf0101b72:     "x %d, y %x, z %d\n"
```

可以看到fmt指针指向的是cprintf()中的格式化字符串。

```bash
vcprintf (fmt=0xf0101b72 "x %d, y %x, z %d\n", ap=0xf010ffd4 "\001") at kern/printf.c:18
18      {
(gdb) x/s 0xf010ffd4
0xf010ffd4:     "\001"
```

而ap指针指向的是第一个参数的地址，也就是栈顶的位置（前面说了参数是从右到左压入栈顶，那栈顶应该是1）。

- List (in order of execution) each call to `cons_putc`, `va_arg`, and `vcprintf`. For `cons_putc`, list its argument as well. For `va_arg`, list what `ap` points to before and after the call. For `vcprintf` list the values of its two arguments.

```bash
(gdb) b cons_putc
Breakpoint 1 at 0xf01003c0: file kern/console.c, line 434.
(gdb) b vcprintf
Breakpoint 2 at 0xf0100a94: file kern/printf.c, line 18.
(gdb) b lib/printfmt.c:75
Breakpoint 3 at 0xf01011de: file lib/printfmt.c, line 75.

vcprintf (fmt=0xf0101b72 "x %d, y %x, z %d\n", ap=0xf010ffd4 "\001")
cons_putc (c=-267379853)
cons_putc (c=-267379852)

```



**4.Run the following code.**

```c
    unsigned int i = 0x00646c72;
    cprintf("H%x Wo%s", 57616, &i);
```

打印出来的字符是He110 World, 57616的十六进制是e110, 而 0x72, 0x6c, 0x64, 0x00对应的ascii字符是‘r’、‘l’、‘d’，而因为机器是一个小端机，所以在内存中的布局是由低到高是0x72, 0x6c, 0x64, 0x00，当作字符串就是"rld\0"，我在末尾加了换行符，所以显示的是：

```bash
6828 decimal is 015254 octal!
x 1, y 3, z 4
He110 World
entering test_backtrace 5
```

**5.In the following code, what is going to be printed after `'y='`? (note: the answer is not a specific value.) Why does this happen?**

```c
    cprintf("x=%d y=%d", 3);
```

看一下打印结果。

```bash
x 3, y -267321364
```

为什么会这样子呢？我们知道参数是从右向左压入栈的，所以栈顶是第一个参数的地址，我们可以单步调试查看%esp寄存器查看栈顶内容。

```bash
=> 0xf010012d <i386_init+131>:  call   0xf0100acf <cprintf>
0xf010012d in i386_init () at kern/init.c:45
45              cprintf("x %d, y %d\n", 3);
(gdb) x /16xw $esp
0xf010ffd0:     0xf0101bae      0x00000003      0xf010ffec      0x00000000
0xf010ffe0:     0x00000000      0x00000000      0x00000000      0x00646c72
0xf010fff0:     0x00000000      0x00010094      0x00000000      0xf010003e
0xf0110000 <entry_pgtable>:     0x00000003      0x00001003      0x00002003      0x00003003
```

内存0x0000003的后一个内存地址的内容是0xf010ffec，说明就是y打印的值，转换成十进制就是-267321364，查看结果验证。

```bash
He110 World
x 3, y -267321364
entering test_backtrace 5
```

**6.Let's say that GCC changed its calling convention so that it pushed arguments on the stack in declaration order, so that the last argument is pushed last. How would you have to change `cprintf` or its interface so that it would still be possible to pass it a variable number of arguments?**

所以说C语言支持可变参数的原因正是因为参数压栈是按照从右到左的顺序。

### 栈

最后这一小节将会探讨x86中C语言怎么样使用栈，而且还会在程序中实现一个有用新内核监视器函数，函数作用是打印栈的回溯：一个嵌套call指令保存的指令指针（IP）的列表，  

> **Exercise 9.** Determine where the kernel initializes its stack, and exactly where in memory its stack is located. How does the kernel reserve space for its stack? And at which "end" of this reserved area is the stack pointer initialized to point to?

我们知道内核加载完成后执行的第一条指令是在kern/entry.S中，我们可以到里面去找找，有没有stack的影子。

```c
relocated:

	# Clear the frame pointer register (EBP)
	# so that once we get into debugging C code,
	# stack backtraces will be terminated properly.
	movl	$0x0,%ebp			# nuke frame pointer

	# Set the stack pointer
	movl	$(bootstacktop),%esp
```

可以看到，在跳转到C代码之前，有两条指令`movl	$0x0,%ebp` 和 `movl	$(bootstacktop),%esp`就是对栈顶指针sp进行设置，所以这就是内核初始化栈的地方。  

然后我们通过再`obj/kern/kernel.asm`中找到上面两条指令.

```bash
	# Clear the frame pointer register (EBP)
	# so that once we get into debugging C code,
	# stack backtraces will be terminated properly.
	movl	$0x0,%ebp			# nuke frame pointer
f010002f:	bd 00 00 00 00       	mov    $0x0,%ebp

	# Set the stack pointer
	movl	$(bootstacktop),%esp
f0100034:	bc 00 00 11 f0       	mov    $0xf0110000,%esp
```

我们可以知道栈顶在`$0xf0110000`, 我们在看回到`kern/entry.S`中的92行。

```
.data
###################################################################
# boot stack
###################################################################
	.p2align	PGSHIFT		# force page alignment
	.globl		bootstack
bootstack:
	.space		KSTKSIZE
	.globl		bootstacktop   
bootstacktop:
```

引导栈的过程中，首先进行了分页对齐，然后设置了栈的大小，为KSTKSIZE，我们可以去头文件`inc/memlayout.h`中找到具体大小。

```c
#define KSTKSIZE	(8*PGSIZE)   		// size of a kernel stack
```

KSTKSIZE是八个页的大小，一个页是4KB，所以KSTKSIZE = 32KB。而栈顶地址为0xf0110000，所以栈底地址是：0xf0110000 - 0x8000(32KB) = `0xf0108000`，所以栈空间是`0xf0108000 ~ 0xf0110000`。  

在x86架构机器中，堆栈指针(`esp` 寄存器)指向正在使用栈的最低位置，任何位置属于栈更低的区域都是可以使用的。  

压入一个数据进栈会涉及两步：1、将减少堆栈指针的值。2、把数据放在堆栈指针所指的地址中。  

弹出一个数据也涉及两步：1、读取堆栈指针所指地址的数据。2、将堆栈指针增加。  

在32位保护模式下，堆栈可以存放32位的数据并且esp的值总是可以被4整除。  

作为对比，`ebp`（基指针）寄存器主要与**调用约定**相关联。在一个C函数的入口，函数的序言代码通常通过将它压入栈中来保存先前函数（caller）的base pointer，然后拷贝当前`esp`的值到`ebp`中,也就是`ebp`先指向当前栈顶，然后在整个函数运行期间持续不变（指的是一定赋值后就不再改变，但`esp`会变）。

如果程序中的所有函数都遵循这个约定，那么在程序执行的任何确定的位置，都可以通过跟踪所保存的`ebp`指针的链来回溯栈，而且还可以精确的知道函数的调用顺序。这个能力十分有用，比如，当一个特定的函数因为错误的参数传递导致了`assert`错误或者`panic`，但是你并不确定是哪个函数传递了这些错误的参数，堆栈回溯就可以帮助你找到这个冲突的函数。

> **Exercise 10.** To become familiar with the C calling conventions on the x86, find the address of the `test_backtrace` function in `obj/kern/kernel.asm`, set a breakpoint there, and examine what happens each time it gets called after the kernel starts. How many 32-bit words does each recursive nesting level of `test_backtrace` push on the stack, and what are those words?
>
> Note that, for this exercise to work properly, you should be using the patched version of QEMU available on the [tools](https://pdos.csail.mit.edu/6.828/2018/tools.html) page or on Athena. Otherwise, you'll have to manually translate all breakpoint and memory addresses to linear addresses.

test_backtrace函数第一次调用是在`i386_init`函数中：

```c
        // Test the stack backtrace function (lab 1 only)
        test_backtrace(5);
f01000fa:       83 c4 14                add    $0x14,%esp         		 ;可以忽略，恢复断点
f01000fd:       6a 05                   push   $0x5                      ;把倒数第一个参数也就是5压入栈中
f01000ff:       e8 3c ff ff ff          call   f0100040 <test_backtrace> ;调用test_backtrace函数
f0100104:       83 c4 10                add    $0x10,%esp                ;恢复断点
```

我们在test_backtrace(5)的地址处打一个断点，进行调试。

```bash
=> 0xf01000ff <i386_init+89>:	call   0xf0100040 <test_backtrace>

(gdb) i r
esp            0xf010ffe0	0xf010ffe0
ebp            0xf010fff8	0xf010fff8

(gdb) x/x 0xf010ffe0
0xf010ffe0:	0x00000005

=> 0xf0100040 <test_backtrace>:	push   %ebp
(gdb) i r
esp            0xf010ffdc	0xf010ffdc
ebp            0xf010fff8	0xf010fff8

(gdb) x/x 0xf010ffdc
0xf010ffdc:	0xf0100104
```

所以调用test_backtrace(5)函数前，将两个32位的数据压入栈顶：  

一个是0x00000005，也就是test_backtrace的参数。  

第二个就是0xf0100104，也就是call指令的下一条指令的地址（保护断点，call指令其实对esp进行了修改）。  

test_backtrace函数的反汇编代码解释：

```c
f0100040 <test_backtrace>:
#include <kern/console.h>

// Test the stack backtrace function (lab 1 only)
void
test_backtrace(int x)
{  
f0100040:       55                      push   %ebp                              //压入调用函数(caller)的ebp,也就是f0100040.
f0100041:       89 e5                   mov    %esp,%ebp                         //将当前%esp存到%ebp，作为栈帧
f0100043:       56                      push   %esi                              //将空串压栈
f0100044:       53                      push   %ebx								 //保存%ebx当前值，防止寄存器状态被破坏
f0100045:       e8 82 01 00 00          call   f01001cc <__x86.get_pc_thunk.bx>  //将栈顶的内容保存到%ebx
f010004a:       81 c3 be 12 01 00       add    $0x112be,%ebx                     //
f0100050:       8b 75 08                mov    0x8(%ebp),%esi                    //取出函数调用的第一个参数（也就是x），保存到%esi
        cprintf("entering test_backtrace %d\n", x);                                   
f0100053:       83 ec 08                sub    $0x8,%esp                         //开辟8字节栈空间给cprintf函数使用
f0100056:       56                      push   %esi                              //将cprintf函数的倒数第一个参数（x）压入栈中
f0100057:       8d 83 f8 06 ff ff       lea    -0xf908(%ebx),%eax                //将cprintf函数的倒数第二个参数（格式化字符串）保存到%eax中
f010005d:       50                      push   %eax                              //将cprintf函数的倒数第二个参数压入栈中。
f010005e:       e8 f6 09 00 00          call   f0100a59 <cprintf>                //调用cprintf函数
        if (x > 0)
f0100063:       83 c4 10                add    $0x10,%esp                        //恢复断点
f0100066:       85 f6                   test   %esi,%esi                         //判断x是否大于0
f0100068:       7f 2b                   jg     f0100095 <test_backtrace+0x55>    //如果x大于0，则调用test_backtrace(x-1)
        else
                mon_backtrace(0, 0, 0);                                          //如果x小于或等于0，那么调用mon_backtrace函数
f010006a:       83 ec 04                sub    $0x4,%esp                         //开辟4字节的栈空间给函数使用。
f010006d:       6a 00                   push   $0x0                              //将mon_backtrace函数的倒数第一个函数压入栈中
f010006f:       6a 00                   push   $0x0								 //将倒数第二个参数压入栈中
f0100071:       6a 00                   push   $0x0                              //将倒数第三个参数压入栈中
f0100073:       e8 1b 08 00 00          call   f0100893 <mon_backtrace>          //调用mon_backtrace函数
f0100078:       83 c4 10                add    $0x10,%esp                        //结束mon_backtrace函数调用，恢复断点
        cprintf("leaving test_backtrace %d\n", x);
f010007b:       83 ec 08                sub    $0x8,%esp                         //开辟8字节的栈空间给cprintf函数使用
f010007e:       56                      push   %esi                              //将cprintf函数的倒数第一个参数x压如栈中
f010007f:       8d 83 14 07 ff ff       lea    -0xf8ec(%ebx),%eax                //将cprintf函数的倒数第二个参数（格式化字符串）保存到%eax
f0100085:       50                      push   %eax                              //将cprintf函数的倒数第二个参数压入到栈中
f0100086:       e8 ce 09 00 00          call   f0100a59 <cprintf>                //调用cprintf函数（callee）

```

不难发现其实C函数的调用机制，如下图  

```bash
high  |            |
  ^   |____________|
  |   |    argN    |
  |   |    arg1    |
  |   |    arg0    |
  |   |  old %eip  |
  |   |  old %ebp  |  <--  %ebp  
  |   | local vars |
  |   | saved regs |  <--- %esp
  v   |------------|
 low  |            |
```



上面的exercise可以得到一些信息，我们要使用这些信息来实现一个栈backtrace函数——mon_backtrace函数。这个函数的原型已经在`kern/monitor.c`为我们写好了，我们可以使用纯C代码来实现。  

也许`inc/x86`头文件中的`read_ebp()`函数会有帮助，同时你需要将实现好的函数挂接到内核监视器的命令列表中，以便用户可以交互的调用。  

回溯函数应该按下面格式显示一个栈帧的列表。

```bash
Stack backtrace:
  ebp f0109e58  eip f0100a62  args 00000001 f0109e80 f0109e98 f0100ed2 00000031
  ebp f0109ed8  eip f01000d6  args 00000000 00000000 f0100058 f0109f28 00000061
  ...
```

每一行包括`ebp`,`eip`和`args`。  

`ebp`值指明函数使用的栈的基指针，例如：在进入函数，函数的序言代码设置的基本指针之后，堆栈指针的位置。其实就是

列出来的`eip`的值是函数返回指令的指针：函数返回时指向的指令地址。返回指令指针通常指向`call`指令的后面那条指令。  

最后五个列出来的十六进制数是讨论函数的前五个参数，它们会在被函数调用前从右到左被压入堆栈中，如果callee少于五个参数，确实，那么所有五个值不一定都有用。（思考，为什么backtrace代码不能检测出实际上有多少个参数？这个限制如何被修正。）  

第一行打印反映了当前正在执行的函数，即`mon_backtrace`本身的命名，第二行打印的是`mon_backtrace` 函数，打印的第三行反映了被调用的函数以此类推。实现应该打印所有的栈帧。  

通过研究`kern/entry.S`可以找到有一个简单的方法来知道什么时候应该停下来。  

> **Exercise 11.** Implement the backtrace function as specified above. Use the same format as in the example, since otherwise the grading script will be confused. When you think you have it working right, run make grade to see if its output conforms to what our grading script expects, and fix it if it doesn't. *After* you have handed in your Lab 1 code, you are welcome to change the output format of the backtrace function any way you like.
>
> If you use `read_ebp()`, note that GCC may generate "optimized" code that calls `read_ebp()` *before* `mon_backtrace()`'s function prologue, which results in an incomplete stack trace (the stack frame of the most recent function call is missing). While we have tried to disable optimizations that cause this reordering, you may want to examine the assembly of `mon_backtrace()` and make sure the call to `read_ebp()` is happening after the function prologue.

```c
int
mon_backtrace(int argc, char **argv, struct Trapframe *tf)
{
        // Your code here.
        uint32_t *ebp;
        cprintf("Stack Backtrace:\n");

        ebp = read_ebp();
        while(ebp) {
            cprintf("ebp %x eip %x args %x %x %x %x %x\n", ebp, ebp[1], ebp[2], ebp[3], ebp[4], ebp[5], ebp[6]);
            ebp = (uin32_t *) *ebp;             //p[0]存放的是旧的eip的值。
        }

        return 0;
}
```

> **Exercise 12.** Modify your stack backtrace function to display, for each `eip`, the function name, source file name, and line number corresponding to that `eip`.
>
> In `debuginfo_eip`, where do `__STAB_*` come from? This question has a long answer; to help you to discover the answer, here are some things you might want to do:
>
> - look in the file `kern/kernel.ld` for `__STAB_*`
> - run objdump -h obj/kern/kernel
> - run objdump -G obj/kern/kernel
> - run gcc -pipe -nostdinc -O2 -fno-builtin -I. -MD -Wall -Wno-format -DJOS_KERNEL -gstabs -c -S kern/init.c, and look at init.s.
> - see if the bootloader loads the symbol table in memory as part of loading the kernel binary
>
> Complete the implementation of `debuginfo_eip` by inserting the call to `stab_binsearch` to find the line number for an address.
>
> Add a `backtrace` command to the kernel monitor, and extend your implementation of `mon_backtrace` to call `debuginfo_eip` and print a line for each stack frame of the form:
>
> ```
> K> backtrace
> Stack backtrace:
>   ebp f010ff78  eip f01008ae  args 00000001 f010ff8c 00000000 f0110580 00000000
>          kern/monitor.c:143: monitor+106
>   ebp f010ffd8  eip f0100193  args 00000000 00001aac 00000660 00000000 00000000
>          kern/init.c:49: i386_init+59
>   ebp f010fff8  eip f010003d  args 00000000 00000000 0000ffff 10cf9a00 0000ffff
>          kern/entry.S:70: <unknown>+0
> K> 
> ```
>
> Each line gives the file name and line within that file of the stack frame's `eip`, followed by the name of the function and the offset of the `eip` from the first instruction of the function (e.g., `monitor+106` means the return `eip` is 106 bytes past the beginning of `monitor`).
>
> Be sure to print the file and function names on a separate line, to avoid confusing the grading script.
>
> Tip: printf format strings provide an easy, albeit obscure, way to print non-null-terminated strings like those in STABS tables. `printf("%.*s", length, string)` prints at most `length` characters of `string`. Take a look at the printf man page to find out why this works.
>
> You may find that some functions are missing from the backtrace. For example, you will probably see a call to `monitor()` but not to `runcmd()`. This is because the compiler in-lines some function calls. Other optimizations may cause you to see unexpected line numbers. If you get rid of the `-O2` from `GNUMakefile`, the backtraces may make more sense (but your kernel will run more slowly).

```c
//kdebug.c	
	stab_binsearch(stabs, &lline, &rline, N_SLINE, addr);
	info->eip_line = stabs[lline].n_desc;
```



```c
int
mon_backtrace(int argc, char **argv, struct Trapframe *tf)
{
    // Your code here.
	uint32_t *ebp;
	struct Eipdebuginfo info;
	cprintf("Stack backtrace:\n");

	ebp = (uint32_t *) read_ebp();
	while (ebp) {
		cprintf("ebp %x  eip %x  args %08.x %08.x %08.x %08.x %08.x\n", ebp, *(ebp + 1), *(ebp + 2), *(ebp + 3), *(ebp + 4), *(ebp + 5), *(ebp + 6));
		debuginfo_eip((uintptr_t) ebp[1], &info);
		cprintf("\t%s:%d: %.*s+%d\n", info.eip_file, info.eip_line, info.eip_fn_namelen, info.eip_fn_name, info.eip_fn_addr);
		ebp = (uint32_t *) *ebp;
	}
	return 0;
}
```

这里需要提一下的就是：

```c
printf("%.*s", length, string)      //打印非空终结符字符串string最大长度（length）个字符
```

至此，这个Lab的内容就全部结束了。

