# File system, Spawn and Shell

## Introduction

这次实验我们会实现一个十分简单的文件系统，仅支持读写磁盘上的文件，它不支持Unix中文件所有权和权限的概念，不支持大多数类Unix系文件中的硬链接、符号链接、时间戳和特殊设备文件。

## File system preliminaries

### On-Disk File System Structure

首先要考虑的是如何表示磁盘上的数据结构，即文件系统应该采用什么样的数据结构来组织数据和元数据。受之前抽象物理内存时采用页数组来表示的提示，这里我们可以简单的用块数组来表示，苏哦一需要将磁盘划分为固定大小的**块**。  

其次，我们需要将块数组划分为两个区域：数据区和inode区。**数据区：**也就是用来记录用户数据的区域，当数据很大时，可能需要多个块来记录。**inode区：**文件系统还必须跟踪每个文件的信息，也就是文件的元数据，例如文件的名字、创建日期，以及文件的数据位于哪个块等，我们把这些元数据称为inode，inode只记录元数据，并不记录用户数据，所以inode区所占的磁盘块数量要远小于数据区。  

JOS不采用inode，而是将文件和目录的元数据存储在目录项中，文件和目录逻辑上包含一系列的数据块，而这些数据快可能是分散在整个磁盘上，并不是连续的，就像虚拟地址空间被映射到不同的物理内存一样。  

#### Sectors and Blocks

大多数磁盘不能按字节粒度进行读写，只能以扇区为单位进行读写，现在能按字节寻址的有NVM。在JOS中，扇区为大小为512B。需要区分的是，扇区大小是磁盘硬件的属性，而块大小是OS使用磁盘的概念，文件系统中块的大小必须是扇区大小的整数倍，如若不然可能需要花费额外的IO，代价比较大。而JOS中的块的大小为4KB，这是为了更方便匹配处理器的页面大小。  

### Superblocks

文件系统通常在磁盘上易于查找的位置(例如最前面或者最后面)保留一些特定的块以便保存描述整个文件系统属性的元数据，例如块大小、磁盘大小、文件系统最新挂载的时间、文件系统最后检查错误的时间等等，这些特定的块称为**super block**。  

JOS只有一块super block, 位于磁盘上的块1，也就是第二块。它的数据结构定义如下：

```c
struct Super {
	uint32_t s_magic;		// Magic number: FS_MAGIC
	uint32_t s_nblocks;		// Total number of blocks on disk
	struct File s_root;		// Root directory node
};
```

块0通常保存boot loaders和分区表，所以文件系统通常不会使用第一块磁盘块。在许多真正的文件系统中会维护多个超级快，它们分布在间隔很大的区域中，这样如果其中之一损坏了，还可以通过其它副本进行使用文件系统，这就是利用副本保证可用性和可靠性。

### File Meta-data

如何描述一个文件的元数据的布局如下：包括文件名，文件的大小，文件的类型(普通文件还是目录)，指向构成文件的直接块和简介块的指针。由于我们不使用inode，元数据直接存储在磁盘的目录项中，简单起见，直接使用`File`结构体代表文件的元数据，因为它会同时出现在磁盘和内存中。`f_direct`数组包含了存储文件前10个磁盘块的空间，我们称为`直接块`。对于大小不超过10 * 4KB = 40KB的小文件，这意味着文件的所有块的块号都装进结构体本身。而对于那些更大的文件，我们需要一个地方来保存文件的其余块号，我们会为文件额外分配一个磁盘块，称为文件的`间接块`,它可以保存4096 / 4 = 1024个额外的磁盘块号。我们的文件系统最大可以支持10 + 1024 = 1034个磁盘块，大小超过4MB。为了支持更大的文件，真实的操作系统通常会引入二级间接块和三级间接块。

![image-20210727172451730](.\figure\image-20210727172451730.png)

```c
// Maximum size of a filename (a single path component), including null
// Must be a multiple of 4
#define MAXNAMELEN	128
// Number of block pointers in a File descriptor
#define NDIRECT		10
// Number of direct block pointers in an indirect block
#define NINDIRECT	(BLKSIZE / 4)

struct File {
	char f_name[MAXNAMELEN];	// filename
	off_t f_size;			// file size in bytes
	uint32_t f_type;		// file type

	// Block pointers.
	// A block is allocated iff its value is != 0.
	uint32_t f_direct[NDIRECT];	// direct blocks
	uint32_t f_indirect;		// indirect block

	// Pad out to 256 bytes; must do arithmetic in case we're compiling
	// fsformat on a 64-bit machine.
	uint8_t f_pad[256 - MAXNAMELEN - 8 - 4*NDIRECT - 4];
} __attribute__((packed));	// required only on some 64-bit machine
```

### Directories versus Regular Files

我们的`File`结构体可以表示一个普通文件或者一个目录，它是由字段`f_type`决定的，文件系统管理普通文件和目录大致是一样的，除了它不会解释普通文件的内容，对于目录文件它会将内容解释为一系列描述当前目录下的文件和子目录的`File`结构体。  

超级块结构体定义中也包含着一个`File`结构体，它保存着文件系统根目录的元数据，它的内容会被解释为描述位于根目录下的文件和目录的`File`结构体数组。而子目录也是于此类推。  

## The File System

本次实验，我们需要负责将磁盘块读入`块缓存(block cache)`和刷新回磁盘、分配磁盘块、将文件偏移量映射到磁盘块和在实现`read`、`write`、`open`IPC接口。

### disk access

文件系统首先需要访问磁盘，相较于宏内核那样把IDE磁盘驱动器直接添加进内核然后添加必要的系统调用去访问它，为了简单起见，我们采用直接将IDE磁盘驱动器作为一个用户态文件系统进程实现。为了让文件系统本身具有实现磁盘访问的权限，我们需要简单修改下内核。  

我们可以通过轮询、程序IO(PIO)而不适应磁盘中断来简单的实现磁盘访问。也可以在用户模式中断驱动设备驱动器，但这样会更加困难因为内核需要现场设备中断并将它们指派到正确的用户进程。  

x86处理器使用`EFLAGS`寄存器中的`IOPL`位来决定用户态的代码是否允许执行设备IO指令例如`IN`和`OUT`指令。因为所有需要访问的IDE磁盘寄存器都位于x86的IO空间中而不是内存映射区域，所以我们唯一需要做的事就是为文件系统进程赋予`IO特权`。ELAGS寄存器中的`IOPL`位就是简单的控制用户模式代码是否可以访问IO空间。

