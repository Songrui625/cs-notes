## Concurrency
本章是介绍golang中的系统编程的并发部分，也是最吸引人的地方。  

### Goroutines
`goroutine`是由Go运行时管理的轻量级线程。  

```go
go f(x, y, z)
```

开启一个新的goroutine运行：
```go
f(x, y, z)
```

`f`, `x`, `y`, `z`的求值发生在当前的goroutine中，并且`f`的执行发生
在新的goroutine中。  

Goroutines运行在同一个地址空间，所以共享内存的访问必须是同步的。`sync`包提供了十分
有帮助的原语，尽管你不会过多的使用它们，因为还有其它的原语。  

```go
package main

import (
	"fmt"
	"time"
)

func say(s string) {
	for i := 0; i < 5; i++ {
		time.Sleep(100 * time.Millisecond)
		fmt.Println(s)
	}
}

func main() {
	go say("world")
	say("hello")
}

```

### Channels
频道是一种管道，你可以用频道运算符`<-`来发送和接收数据。  

```go
ch <- v // Send v to channel ch.
v := <-ch // Receive from ch, and assign value to v
```

(数据在箭头的方向进行流动)  

就像map和切片，频道必须先创建再使用。  

```go
ch := make(chan int)
```

默认的，发送和接收会进入阻塞状态直到另一端准备好。这种特性让goroutines可以不通过显示的锁
和条件变量进行同步。  

样例代码对一个切片种的元素进行求和。再两个gorountines中分布式的工作。一旦两个gorountines完成了
它们的运算，它就会计算最后结果。

```go
package main

import "fmt"

func sum(s []int, c chan int) {
	sum := 0
	for _, v := range s {
		sum += v
	}
	c <- sum // send sum to c
}

func main() {
	s := []int{7, 2, 8, -9, 4, 0}

	c := make(chan int)
	go sum(s[:len(s)/2], c)
	go sum(s[len(s)/2:], c)
	x, y := <-c, <-c // receive from c

	fmt.Println(x, y, x+y)
}
```

### Buffered Channels
频道可以被缓冲。向`make`提供一个缓冲区长度作为第二个参数可以初始化一个缓冲频道。  

```go
ch := make(chan int, 100)
```

当缓冲区慢的时候，向一个缓冲频道发送数据会进入阻塞。当缓冲区空时，接收会进入阻塞。  

修改样例代码来填满缓冲区看看会发生什么？  

```go
package main

import "fmt"

func main() {
    ch := make(chan int, 2)
    ch <- 1
    ch <- 2
    fmt.Println(<-ch)
    fmt.Println(<-ch)
}
```

### Range and Close
一个发送者可以`close`管道，这表明没有更多的数据需要发送了。接收者可以通过赋值给接收者表达式
的第二个参数测试一个管道是否已经被关闭。  

```go
v, ok := <-ch
```

如果没有更多的值可以接收并且管道已经关闭，`ok`是`false`。  

循环 `for i := range c`会重复的从管道接收数据直到它被关闭。  

**注意:** 只有发送方才应该关闭管道，而不是接收方。在一个关闭的管道上发送数据会造成恐慌。  

**另一个注意:**管道不像文件；你不需要经常关闭他们。当接收方被告知，不会再有数据到来的时候，关闭
是必要的，例如终结一个`range`循环。  

```go
package main

import (
	"fmt"
)

func fibonacci(n int, c chan int) {
	x, y := 0, 1
	for i := 0; i < n; i++ {
		c <- x
		x, y = y, x+y
	}
	close(c)
}

func main() {
	c := make(chan int, 10)
	go fibonacci(cap(c), c)
	for i := range c {
		fmt.Println(i)
	}
}
```

### Select
`select`语句让goroutine等待多个通信操作。  

`select`会进入阻塞直到其中一个case可以运行，然后它开始执行那个case。如果多个case都准备好运行了，
它会选择其中一个。  

```go
package main

import "fmt"

func fibonacci(c, quit chan int) {
	x, y := 0, 1
	for {
		select {
		case c <- x:
			x, y = y, x+y
		case <-quit:
			fmt.Println("quit")
			return
		}
	}
}

func main() {
	c := make(chan int)
	quit := make(chan int)
	go func() {
		for i := 0; i < 10; i++ {
			fmt.Println(<-c)
		}
		quit <- 0
	}()
	fibonacci(c, quit)
}

```

### Default Selection
`select`中的`default`会在没有其他case准备好时运行。  

使用`default`case 尝试非阻塞的发送或者接收。
```go
select {
case i := <-c:
    // use i
default:
    // receiving from c would block
}
```

### Exercise: Equivalent Binary Trees
有许多不同的二叉树，其中存储了相同的值序列。例如，下面是一个存储了`1, 1, 2, 3, 5, 8, 13`序列的两个二叉树。
```
      3                  8
    /   \               / \
   1     8             3   13
  / \   / \           / \
 1   2 5   13        1   5
                    / \
                   1   2  
```
在大多数编程语言中检查两个二叉树是否存储相同的序列的函数是十分复杂的。我们会使用Go的并发和管道来编写一个
简单的答案。  

这个样例使用了`tree`包，定义了如下类型：
```go
type tree struct {
    Left *Tree
    Value int
    Right *Tree
}
```

1、实现`Walk`函数  

2、测试`Walk`函数  

函数`tree.New(k)`会构建一个随机结构(但总是有序)的二叉树，保存值为`k`,`2k`,`3k`,...,`10k`  

创建一个新的管道`ch`然后开始遍历：
```go
go walk(tree.New(1), ch)
```
然后读取并打印10个值。它可能会是：1,2,3,...,10  

3、使用`Walk`实现`Same`函数来判断`t1`和`t2`是否存储相同的值。  

4、测试`Same`函数。  
`Same(tree.new(1), Tree.new(1))`应该返回真，并且`Same(tree.New(1), Tree.New(2))`应该返回假。  

```go
package main

import (
	"golang.org/x/tour/tree"
	"fmt"
)

// Walk walks the tree t sending all values
// from the tree to the channel ch.
func Walker(t *tree.Tree, ch chan int) {
    if t == nil {
        return
    }
    Walker(t.Left, ch)
    ch <- t.Value
    Walker(t.Right, ch)
}

func Walk(t *tree.Tree, ch chan int) {
	Walker(t, ch)
	close(ch)
}

// Same determines whether the trees
// t1 and t2 contain the same values.
func Same(t1, t2 *tree.Tree) bool {
    ch1 := make(chan int)
    ch2 := make(chan int)

    // Walk(t1, ch1)
    // Walk(t2, ch2)
    go Walk(t1, ch1)
    go Walk(t2, ch2)
	for {
		x, ok1 := <- ch1
		y, ok2 := <- ch2
		
		if x != y || ok1 != ok2 {
			return false
		}
		
		if !ok1 || !ok2 {
			break;
		}
	}
    return true
}

func main() {
    fmt.Println(Same(tree.New(1), tree.New(1)))
    fmt.Println(Same(tree.New(1), tree.New(2)))
}

```

原来一直通过不了，原来是
```go
    go Walk(t1, ch1)
    go Walk(t2, ch2)
```
而不是
```go
    Walk(t1, ch1)
    Walk(t2, ch2)
```