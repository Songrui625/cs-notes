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

