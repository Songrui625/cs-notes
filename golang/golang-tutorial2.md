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