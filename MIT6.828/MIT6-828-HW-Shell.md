---
title: 'MIT6.828 HW:Shell'
date: 2021-01-07 20:39:06
tags: xv6

---

这个作业的目的是熟悉Unix 系统调用接口，以及通过实现一些shell的特性来熟悉shell。

<!--more-->

**Prerequisities:** 阅读xv6 book的第0章节  

下载6.828 shell，浏览一遍。6.828 shell包含了两个主要部分：**1、解析shell命令**， **2、执行它们**。

## Executing simple commands

实现简单的命令，如

```bash
$ ls
```

作业已经为我们提供了execcmd函数，我们只需要完成`runcmd()`中的 `case ' ' `。man手册的 exec会有所帮助， 阅读execv相关的内容。如果执行失败打印错误信息。

```c
  case ' ':
    ecmd = (struct execcmd*)cmd;
    if(ecmd->argv[0] == 0)
      _exit(0);
    // fprintf(stderr, "exec not implemented\n");
    // Your code here ...
    r = execv(ecmd->argv[0], ecmd->argv);
    if (!r)
        fprintf(stderr, "Unknown command:%s\n", ecmd->argv[0]);
    break;
```

这里需要注意的点就是, `execv(const char *path, char **argv)`， 第二个参数argv是包括执行文件名的。  

但是如果只是这样一句代码的话，是无法实现到系统环境变量搜索文件的。所以我们需要更改一下。

```c
  char *root = "/bin/";
  char *usr_root = "/usr/bin/";
  char *path = (char *) malloc(100 * sizeof(char)); // 一定要有足够的空间。
  ...
  case ' ':
    ecmd = (struct execcmd*)cmd;
    if(ecmd->argv[0] == 0)
      _exit(0);
    // fprintf(stderr, "exec not implemented\n");
    // Your code here ...
    // printf("ecmd->argv[0]:%s\n", ecmd->argv[0]);
    if (execv(ecmd->argv[0], ecmd->argv) < 0) {
      //path doesn't in current dir.
      strcpy(path, root);
      strcat(path, ecmd->argv[0]);
      if (execv(path, ecmd->argv) < 0) {
        //path doesn't in /usr
        strcpy(path, usr_root);
        strcat(path, ecmd->argv[0]);
        if (execv(path, ecmd->argv) < 0)
          //path doesn't in /usr/bin as well
          fprintf(stderr, "unknown command:%s\n", ecmd->argv[0]);
      }
    }
    break;
```

## I/O redirection

实现IO重定向，让你的程序可以运行如下命令：

```bash
echo "6.828 is cool" > x.txt
cat < x.txt
```

作业同样为我们提供了骨架代码，我们只需填空就好了, man手册的open和close系统调用会有所帮助。  

**关键知识: **close系统调用会释放一个已经分配的分拣描述符，以便后面open、dup或者pipe系统调用可重用。新分配的文件描述符总是当前进程中**最小的未使用**描述符。

```c
  case '>':
  case '<':
    rcmd = (struct redircmd*)cmd;
    // fprintf(stderr, "redir not implemented\n");
    // Your code here ...
    close(rcmd->fd);
    if ((r = open(rcmd->file, rcmd->flags, S_IWUSR | S_IRUSR)) < 0) {
      fprintf(stderr, "open failed.\n");
      exit(0);
    }
    runcmd(rcmd->cmd);
    break;
```

我们来看下 `echo "6.828 is cool" > x.txt`是如何把输出重定向到x.txt中的。  

1、parsecmd对命令进行解析，封装成了cmd，然后执行rumcmd。  

2、对cmd里的type进行识别，由于是 >，跳到了上述函数。  

3、rcmd->fd是1， 此时fd是归shell进程所有，调用close系统调用释放文件描述符。(思考题：为什么fd的值不可能是0？)  

4、调用open系统调用， 返回了一个文件描述符，数值为1，此时fd所指为open的文件。那么当我们向这个fd写入内容时，比如说print，那么就会向文件中写入内容。  

5、此时会调用runcmd函数，执行echo "6.828 is cool", 字符串会被打印到文件描述符中，注意：此时文件描述符指向的是文件x.txt。所以字符串会被写入到x.txt中。  

## Implement pipes

实现IO重定向，让你的程序可以运行如下命令：

```bash
$ ls | sort | uniq | wc
```

作业提供好了骨架代码，我们只需填空。man手册中的pipe， fork， close和dup会有所帮助。可以参考xv6源码sh.c, 加以理解。

```c
  case '|':
    pcmd = (struct pipecmd*)cmd;
    // fprintf(stderr, "pipe not implemented\n");
    // Your code here ...
    int p[2];
    if (pipe(p) < 0) {
      perror("pipe");
      exit(0);
    }
    if (fork1() == 0) {
      // left process
      //release fd so that it can reuse later
      close(1);
      dup(p[1]); // ret is 1, and it refer to the p[1](write site of pipe)
      close(p[0]);
      close(p[1]);
      runcmd(pcmd->left);
    }

    if (fork1() == 0) {
      //right process
      close(0);
      dup(p[0]); // ret is 0, and it refer to the p[0](read site of pipe)
      close(p[0]);
      close(p[1]);
      runcmd(pcmd->right);
    }
    close(p[0]);
    close(p[1]);
    wait(&r);
    wait(&r);
    break;
```

管道只能实现单向通信，要实现双向通信必须创建两个管道。  

并且，不使用的读端和写端必须关闭。原因是：1、防止浪费文件描述符，以便重用。2、可以检测到EOF。





