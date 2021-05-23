本篇是官方教程的第二章,主要学习Go中的**Method**和**Interfaces**

### Methods
Go语言没有`class`关键字，这就意味着我们不能在类中定义函数来操作类中的成员变量。但是我们可以在类型上
定义`methods`，它能达到上述的效果。  

`method`是带有特殊`接收者`参数的函数。  

`接收者`出现在`func`关键字和和`method`名之间。  

在下面例子中，`Abs` `method`拥有一个类型为`Vertex`的`接收者``v`。

```go
package main

import(
    "fmt"
    "math"
)

type Vertex struct {
    X, Y float64
}

func (v Vertex) Abs() float64 {
    return math.Sqrt(v.X * v.X + v.Y * v.Y)
}

func main() {
    v := Vertex{3, 4}
    fmt.Println(v.Abs())
}
```
上面这段代码在c++中表示如下：
```c++
#include <bis/stdc++.h>

class Vertex {
public:
    Vertex() = default;
    Vertex(float x, float y): x_(x), y_(y) {}

    ~Vertex() = default;

    float Abs() {
        return this->x_ * this->x_ + this->y_ * this->y_;
    }

private:
    float x_{0};
    float y_{0};
}

int main() {
    Vertex *v = new Vertex(3, 4);
    std::cout << v->Abs() << endl;

    return 0; 
}
```

### Methods are functions
**记住**：`method`仅仅是带有一个`接收者`参数的函数。  

下面是等价的常规`Abs`函数写法:
```go

func Abs(v Vertex) float64 {
    return math.Sqrt(v.X * v.X + v.Y * v.Y)
}

```

### Methods continued
我们还可以在非结构体类型上声明一个`method`  

在下面的例子中，我们会看到一个数值类型`MyFloat`有一个`Abs``method`

我们**仅**可以**在同一个`package`下**定义的类型`接收者`上声明`method`

```go
packgae main

import(
    "fmt"
    "math"
)

type MyFloat float64

func (f MyFloat) Abs() float64 {
    if f < 0 {
        return float64(-f)
    }
    return float64(f)
}

func main() {
    f := MyFloat(-math.Sqrt(2))
    fmt.Println(f.Abs())
}
```

### Pointer receivers
我们可以声明带有指针`接收者`的`methods`  

这意味着对于某些类型`T`, `接收者`类型可以拥有字面值语法`*T`(同样，`T`本身不能是`*int`这样的指针)  

例如，`Scale method`定义在`*Vertex`上。  

带有指针`接收者`的`Methods`可以修改指针所指向的对象的值。因为，`methods`通常需要修改他们的接收者，指针接收者通常更常用，而不是那些值接收者。  

尝试删除`Scale`函数声明中的 `*`，观察程序的行为会如何变化。  

一个值接收者，`Scale method`会在原始的拷贝上进行操作，这和其他语言的函数参数一样。  

```go
package main

import(
    "fmt"
    "math"
)

typef Vertex struct {
    X, Y float64
}

func (v Vertex) Abs() float64 {
    return math.Sqrt(v.X * v.X + v.Y * v.Y)
}

func (v *Vertex) Scale(f float64) {
    v.X = v.X * f
    v.Y = v.Y * f
}

func main() {
    v := Vertex{3, 4}
    v.Scale(10)
    fmt.Println(v.Abs())
}

```

### Pointers and functions
我们看下下面用函数重写的`Abs`和`Scale`方法。  

同样的，尝试在函数定义中删除`*`。为什么行为会改变？我们应该做怎么样的改变才能编译下面的例子？

```go

package main

import(
    "fmt"
    "math"
)

type Vertex struct {
    X, Y float64
}

func Abs(v Vertex) float64 {
    return math.Sqrt(v.X * v.X + v.Y * v.Y)
}

func Scale(v *Vertex, f float64) {
    v.X = v.X * f
    v.Y = v.Y * f
}

func main() {
    v := Vertex{3, 4}
    Scale(&v, 10)
    fmt.Println(Abs(v))
}

```

### Methods and pointer indirection
比较先前的两个程序，我们可能注意到了带有指针参数的函数必须接收一个指针：

```go
var v Vertex
ScaleFunc(v, 5) // Compile error!
ScaleFunc(&v, 5) // OK
```

当带有指针接收者`methods`被调用时接收一个值或者一个指针作为接收者：

```go
var v Vertex
v.Scale(5) // OK
p := &v
p.Scale(10) // OK
```
对于语句`v.Scale(5)`，即使`v`时一个值而不是一个指针，带有指针参数的`method`也可以被自动调用。
这是因为，为了方便起见，因为`Scale method`有一个指针接收者，它会把语句`v.Scale(5)`解释为`(&v).Scale(5)`

```go
package main

import "fmt"

type Vertex struct {
	X, Y float64
}

func (v *Vertex) Scale(f float64) {
	v.X = v.X * f
	v.Y = v.Y * f
}

func ScaleFunc(v *Vertex, f float64) {
	v.X = v.X * f
	v.Y = v.Y * f
}

func main() {
	v := Vertex{3, 4}
	v.Scale(2)
	ScaleFunc(&v, 10)

	p := &Vertex{4, 3}
	p.Scale(3)
	ScaleFunc(p, 8)

	fmt.Println(v, p)
}
```

### Methods and pointerr indirection(2)
同等的事情发生在相反的方向。

```go
var v Vertex
fmt.Println(AbsFunc(v))  // OK
fmt.Println(AbsFunc(&v)) // Compile error!
```

而具有值接收者的方法在被调用时采用值或指针作为接收者：
```go
var v Vertex
fmt.Println(v.Abs()) // OK
p := &v
fmt.Println(p.Abs()) // OK
```

在这种情况下，`method`调用`p.Abs()`会被解释为`(*p).Abs()`

### Choosing a value or pointer receiver
有两种情况下使用指针接收者。  

第一、`method`能修改接收者指向的对象。  

第二就是在每次`method`调用时避免拷贝值。这样在接收者是一个很大的结构体时十分高效。  

在这个例子中，`Scale`和`Abs`都带有一个接收者类型`* Vertex`，即使`Abs method`不需要修改它的接收者。  

通常来说，所有的`methods`要么接收值接收者，要么接收指针接收者，但不能是两者的混合(后面会知道为什么)。  

```go
package main

import (
	"fmt"
	"math"
)

type Vertex struct {
	X, Y float64
}

func (v *Vertex) Scale(f float64) {
	v.X = v.X * f
	v.Y = v.Y * f
}

func (v *Vertex) Abs() float64 {
	return math.Sqrt(v.X*v.X + v.Y*v.Y)
}

func main() {
	v := &Vertex{3, 4}
	fmt.Printf("Before scaling: %+v, Abs: %v\n", v, v.Abs())
	v.Scale(5)
	fmt.Printf("After scaling: %+v, Abs: %v\n", v, v.Abs())
}
```

### Interfaces
一个`interface`类型定义被视为`method`签名的集合。  

接口类型的值可以包含实现那些方法的任何值。

**注意**：在示例代码的22行有个错误。`Vertex`(值类型)没有实现`Abser`因为`Abs`方法仅仅在`*Vertex`上
定义。  

```go
package main

import (
	"fmt"
	"math"
)

type Abser interface {
	Abs() float64
}

func main() {
	var a Abser
	f := MyFloat(-math.Sqrt2)
	v := Vertex{3, 4}

	a = f  // a MyFloat implements Abser
	a = &v // a *Vertex implements Abser

	// In the following line, v is a Vertex (not *Vertex)
	// and does NOT implement Abser.
	a = v

	fmt.Println(a.Abs())
}

type MyFloat float64

func (f MyFloat) Abs() float64 {
	if f < 0 {
		return float64(-f)
	}
	return float64(f)
}

type Vertex struct {
	X, Y float64
}

func (v *Vertex) Abs() float64 {
	return math.Sqrt(v.X*v.X + v.Y*v.Y)
}
```

### Interfaces are implemented implicitly
类型实现接口可以通过实现它的`methods`。没有明确的声明意图，没有"implements"关键字。  

隐式接口对一个接口的定义和实现进行了解耦，实现可以在没有预先安排的情况下出现在任何package中。  

```go
package main

import "fmt"

type I interface {
    M()
}

type T struct {
    S string
}

// this method means T implements the  interface I,
// but we don't need to explicityly declare that it does so
func (t T) M() {
    fmt.Println(t.S)
}

func main() {
    var i I = T{"hello"}
    i.M()``
}

```

### Interface values

在幕后，接口值可以被认为时一个值元组和具体类型：

```go
(value, type)
```

一个接口值保存着一个特定的基础具体类型的值。

调用一个接口值的`method`会执行它的基本类型同名的`method`

```go
package main

import(
    "fmt"
    "math"
)

type I interface {
    M()
}

type T struct {
    S string
}

func (t *T) M() {
    fmt.Println(t.S)
}

type F float64

func (f F) M() {
    fmt.Println(f)
}

func main() {
    var i I

    i = &T{"Hello"}
    describe(i)
    i.M()

    i = F(math.Pi)
    describe(i)
    i.M()
}

func describe(i I) {
    fmt.Println("(%v, %T)", i, i)
}
```

### Interface values with nil underlying values
如果接口本身的具体值是`nil`，`method`会被调用，参数是`nil`接收者  

在某些编程语言中，这会触发一个空指针异常，但是在Go中编写处理具有
`nil`接收者参数的`method`是很常见的。(与下面例子中`method``M`一样)  

注意，包含`nil`具体值的接口本身值是`non-nil`的。  

```go
package main

import "fmt"

type I interface {
	M()
}

type T struct {
	S string
}

func (t *T) M() {
	if t == nil {
		fmt.Println("<nil>")
		return
	}
	fmt.Println(t.S)
}

func main() {
	var i I

	var t *T
	i = t
	describe(i)
	i.M()

	i = &T{"hello"}
	describe(i)
	i.M()
}

func describe(i I) {
	fmt.Printf("(%v, %T)\n", i, i)
}
```

### Nil interface values
一个nil接口值保存既不保存值也不保存具体类型。  

在一个nil接口上调用方法是一个运行时错误因为在接口元组中没有表示调用具体方法的类型。  

```go
package main

import "fmt"

type I interface {
    M()
}

func main() {
    var i I
    describe(i)
    i.M()
}

func describe(i I) {
    fmt.Println("(%v, %T)\n", i, i)
}
```

### The empty interface

指定0个方法的接口类型也被称为空接口。
```go
interface {}
```

一个空接口可能保存任何类型的值。(每个类型至少实现0个方法)  

空接口常被用于处理未知类型值的代码。例如，`fmt.Print`接收任何类型`interface{}`参数的值。

```go
package main

import "fmt"

func main() {
	var i interface{}
	describe(i)

	i = 42
	describe(i)

	i = "hello"
	describe(i)
}

func describe(i interface{}) {
	fmt.Printf("(%v, %T)\n", i, i)
}
```

### Type assertions
一个类型断言提供了访问一个接口值的基础具体值的方法。  

```go
t := i.(T)
```

这个语句断言接口值`i`保存着具体类型`T`并将基础类型`T`值赋值给变量`t`  

如果`i`没有保存一个`T`类型的值，那么语句会触发一个恐慌。  

为了测试接口值是否保存一个具体的类型，类型断言可以返回两个值：基础值和
一个报告断言是否成功布尔值。
```go
t, ok := i.(T)
```
如果`i`保存着类型`T`，那么`t`将会是基础值，并且`ok`会是true。
如果不是，`ok`会是false并且`t`会被赋值为类型`T`的0值，并且不会发生恐慌。  

注意这个语法和从map中读取数据语法的相似性。  

```go
package main

import "fmt"

func main() {
    var i interface{} = "hello"

    s := i.(string)
    fmt.Println(s)

    s, ok := i.(string)
    fmt.Println(s, ok)

    f, ok := i.(float64)
    fmt.Println(f, ok)

    f = i.(float64) // panic
    fmt.Println(f)
}
```

### Type switches

类型开关是一种允许串联几个类型断言的构造。  

类型开关就像是个正常开关语句，但是类型开关指定类型而不是值，并且那些值被用来与接口给定的值
进行比较。  

```go

switch v := i.(type) {
    case T:
    // here v has type T
    case S:
    // here v has type S
    default:
    // no match; here v has the same type  as i
}

```

类型选择的声明语法和类型断言一样`i.(T)`，但是指定类型`T`需要用关键字`type`。

选择语句测试接口值`i`是否保存着类型`T`或者`S`的值。在每个`T`和`S`的例子种，变量`v`将会是类型`T`
或者类型`S`类型并且保存着`i`的值。在默认的例子中(没有匹配的值)，变量`v`和`i`具有同样的接口类型和值。  

```go
package main

import "fmt"

func do(i interface{}) {
	switch v := i.(type) {
	case int:
		fmt.Printf("Twice %v is %v\n", v, v*2)
	case string:
		fmt.Printf("%q is %v bytes long\n", v, len(v))
	default:
		fmt.Printf("I don't know about type %T!\n", v)
	}
}

func main() {
	do(21)
	do("hello")
	do(true)
}

```

### Stringers
最普遍的接口之一就是由`fmt`包定义的`Stringer`。
```go
type Stringer interface {
    String() string
}
```

`Stringer`是将自身作为字符串描述的类型。`fmt`包会查找这个接口来打印值。类似于Java中的toString()。  

```go
package main

type Person struct {
    Name string
    Age int
}

func (p Person) String() string {
    return fmt.Println("%v (%v years)", p.Name, p.Age)
}

func main() {
	a := Person{"Arthur Dent", 42}
	z := Person{"Zaphod Beeblebrox", 9001}
	fmt.Println(a, z)
}
```

### Exercise: Stringers
让`IPAddr`类型实现`fmt.Stringer`接口来打印IP地址。  

例如，`IPAddr{1,2,3,4}` 应该打印出"1.2.3.4"  

```go
package main

import "fmt"

type IPAddr [4]byte

// TODO: Add a "String() string" method to IPAddr.

func (ip IPAddr) String() string {
	return fmt.Sprintf("%v.%v.%v.%v", ip[0], ip[1], ip[2], ip[3])
}

func main() {
	hosts := map[string]IPAddr{
		"loopback":  {127, 0, 0, 1},
		"googleDNS": {8, 8, 8, 8},
	}
	for name, ip := range hosts {
		fmt.Printf("%v: %v\n", name, ip)
	}
}

```

### Errors
G哦程序通过`error`值来表达错误。  

`error`类型是一个类似于`fmt.Stringer`的内置接口。  
```go
type error interface {
    Error() string
}
```

(和`fmt.Stringer`一样，`fmt`包会查找`error`接口来打印值)  

函数通常会返回一个`error`值，并调用代码通过错误是否等于`nil`来处理错误。  

```go
i, err := strconv.Atoi("42")
if err != nil {
    fmt.Printf("couldn't convert number: %v\n", err)
    return
}
fmt.Println("Converted integer:", i)
```

一个nil的`error`表示成功；一个非nil的`error`表示失败。  

```go
package main

import (
	"fmt"
	"time"
)

type MyError struct {
	When time.Time
	What string
}

func (e *MyError) Error() string {
	return fmt.Sprintf("at %v, %s",
		e.When, e.What)
}

func run() error {
	return &MyError{
		time.Now(),
		"it didn't work",
	}
}

func main() {
	if err := run(); err != nil {
		fmt.Println(err)
	}
}
```

### Exercise: Errors
从之前的练习拷贝你的`Sqrt`函数，然后修改它返回一个`error`值。  

当给定了一个负数时`Sqrt`应该返回一个非空的错误，因为它不支持复数。  

创建一个新的类型：
```go
type ErrNegativeSqrt float64
```

让他成为一个错误:
```go
func (e ErrNegativeSqrt) Error() string
```

方法`ErrNegativeSqrt(-2).Error()`返回`"cannot Sqrt negative number: -2"`。  

注意：在`Error`方法中调用`fmt.Sprint(e)`会让程序进入一个无限循环。我们可以通过首先将`e`转换
成:`fmt.Sprint(float64(e))`。为什么？  

改变我们的`Sqrt`函数，当给定一个负数时，返回一个`ErrNegativeSqrt`值。  

```go
package main

import (
	"fmt"
	"math"
)

type ErrNegativeSqrt float64

func (e ErrNegativeSqrt) Error() string {
	return fmt.Sprintln("cannot Sqrt negative number: %v", e)
}

func Sqrt(x float64) (float64, error) {
	var e error
	if x < 0 {
		return x, ErrNegativeSqrt(x)
	}
	delta := 0.00001
	z := float64(1)
	for math.Abs(z * z - x) > delta {
		z -= (z * z - x ) / ( 2 * z)
	}
	return z, e
}

func main() {
	fmt.Println(Sqrt(2))
	fmt.Println(Sqrt(-2))
}

```

### Readers
`io`包制定了`io.Reader`接口，代表数据流的读取端。  

Go标准库包含许多这个接口的实现，包括分拣、网络连接、压缩机、密码和其他。  

`io.Reader`接口又一个`Read`方法：
```go
func (T) Read(b []byte) (n int, err error)
```

`Read`输入给定数据的字节切片并且返回成功输入的字节数和一个错误值。当流结束的时候，它返回一个`io.EOF`。  

样例代码创造了一个`strings.Reader`并且每次消耗8个字节。  

```go
package main

import (
    "fmt"
    "io"
    "strings"
)

func main() {
    r := strings.NewReader("Hello, Reader!")

    b := make([]byte, 8)

    for {
        n, err := r.Read(b)
        fmt.Printf("n = %v err = %v b = %v\n", n, err, b)
        fmt.Printf("b[:n] = %q\n", b["n])
        if err == io.EOF {
            break
        }
    }
    
}
```

### Exercise: Readers
实现一个可以发射一个ASCII字符'A'的无限流`Reader`类型。  

```go
package main

import "golang.org/x/tour/reader"

type MyReader struct {}

// TODO: Add a Read([]byte) (int, error) method to MyReader.
func (m MyReader) Read(bytes []byte) (int, error){
	for i := range bytes {
		bytes[i] = 65
	}
	return len(bytes), nil
}

func main() {
	reader.Validate(MyReader{})
}
```