### 启程：x86汇编

熟悉x86汇编的话完全可以略过这一部分。如果想学习基本的x86汇编，讲义也给出了一个比较不错的入门文章：[Brennan's Guide to inline Assembly](http://www.delorie.com/djgpp/doc/brennan/brennan_att_inline_djgpp.html). 

### 模拟x86

 进入到lab文件夹，进行编译*make*.  

```bash
linux% cd lab
linux% make
+ as kern/entry.S
+ cc kern/entrypgdir.c
+ cc kern/init.c
+ cc kern/console.c
+ cc kern/monitor.c
+ cc kern/printf.c
+ cc kern/kdebug.c
+ cc lib/printfmt.c
+ cc lib/readline.c
+ cc lib/string.c
+ ld obj/kern/kernel
+ as boot/boot.S
+ cc -Os boot/main.c
+ ld boot/boot
boot block is 380 bytes (max 510)
+ mk obj/kern/kernel.img
```

编译期间可能会出现一系列的错误。关于这些错误的解决方案可以参考这个大佬的环境搭建教程：[6.828-JOS-环境搭建](https://www.cnblogs.com/gatsby123/p/9746193.html)

tips:这个编译需要提前安装好qemu模拟器，关于qemu模拟器的安装可以参考上面的大佬的环境搭建。  

如果你编译过程中出现了"undefined reference to `__udivdi3'"错误，是因为你的linux系统是64位的，那么可能需要32位库才能编译成功。  

终端运行如下命令：  

```bash
apt-get install gcc-multilib
```

编译成功之后运行如下命令：*make qemu*如果成功将会看到如下提示。

```bash
$ sudo make qemu
qemu-system-i386 -drive file=obj/kern/kernel.img,index=0,media=disk,format=raw -serial mon:stdio -gdb tcp::25000 -D qemu.log 
VNC server running on `127.0.0.1:5900'
6828 decimal is XXX octal!
entering test_backtrace 5
entering test_backtrace 4
entering test_backtrace 3
entering test_backtrace 2
entering test_backtrace 1
entering test_backtrace 0
leaving test_backtrace 0
leaving test_backtrace 1
leaving test_backtrace 2
leaving test_backtrace 3
leaving test_backtrace 4
leaving test_backtrace 5
Welcome to the JOS kernel monitor!
Type 'help' for a list of commands.
K>
```

这样我们就成功的将内核从硬盘引导起来了。

### The ROM BIOS

我们在linux打开两个终端，进入lab文件夹。第一个运行*make qemu-gdb*命令，显示如下：

```bash
$ sudo make qemu-gdb
***
*** Now run 'make gdb'.
***
qemu-system-i386 -drive file=obj/kern/kernel.img,index=0,media=disk,format=raw -serial mon:stdio -gdb tcp::25000 -D qemu.log  -S
VNC server running on `127.0.0.1:5900'
```

第二个运行 *make gdb*命令，显示如下：

```bash
$ sudo make gdb
gdb -n -x .gdbinit
GNU gdb (Ubuntu 9.1-0ubuntu1) 9.1
Copyright (C) 2020 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word".
+ target remote localhost:25000
warning: No executable has been specified and target does not support
determining executable automatically.  Try using the "file" command.
warning: A handler for the OS ABI "GNU/Linux" is not built into this configuration
of GDB.  Attempting to continue with the default i8086 settings.

The target architecture is assumed to be i8086
[f000:fff0]    0xffff0: ljmp   $0xf000,$0xe05b
0x0000fff0 in ?? ()
```

我们重点关注这一行:

```bash
[f000:fff0]    0xffff0: ljmp   $0xf000,$0xe05b
```

[f000:fff0]就是(cs:ip)的值称为逻辑地址，它是由**段地址：偏移地址**表示的，而0xffff0也就是在内存中实际的**物理地址**，它的值是由**段地址 * 16 + 偏移地址**得来的，也就是 `f000 * 16 + fff0 = f0000 + fff0 = ffff0`.  

为什么是*16呢？这是因为8086、8088CPU的寄存器是16位的，而地址线有20线，16位线的地址如何拼接成20位地址呢？结论就是将段寄存器中的值左移4位，形成20位段基地址，然后和16位偏移地址相加，就得到了物理地址。  

回到实际的物理地址：0xffff0,这是BIOS（0x100000）结束前的16个字节。也就是说，我们现在在运行BIOS，因为当前指令的地址存放的是BIOS的代码和数据，当BIOS运行时，它会创建一个中断描述符表以及初始化不同的设备比如VGA显示，这也是我们能在QEMU窗口看到“Starting SeaBIOS”的这条消息的原因。  

BIOS会初始化PCI总线和所知的所有重要设备后，它将搜索可引导的设备比如软盘、硬盘驱动器，CD-ROM。最终，当他找到一个可引导磁盘时，会从磁盘读取引导加载程序并将控制权转移给该引导加载程序。  

到此我们这一部分告一段落，下一部分就是关于引导加载程序如何引导内核。

