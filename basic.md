## go语言基础

### 1 前言

一门编程语言的基础知识一般包括以下几个部分：

* Hello World
* 基本数据类型
* 流程控制(条件if和switch、循环for)
* 数组和对象(字典)
* 函数
* 输入输出
* 异常处理

### 2 Hello World

``` go
package main

import (
    "fmt"
)

func main() {

	fmt.Println("Hello World")
}
```

### 3 基本数据类型

语言的基本数据类型包含类型本身以及如何定义和使用。

数据类型一般会包含有整型、浮点型、字符、字符串：

go语言中整型分成两类，一类是占用空间与机器字长相关的int和uint，另一类是将占用空间表示在类型中的int8、int16、int32、int64以及相应的uint8、uint16、uint32、uint64。

go语言中浮点型有float32和float64。

go语言中字符型相关的有两个类型：byte和rune，它们分别是uint8和int32的别名。byte只能用于表示8位比特位，rune则可以用来表示一个unicode字符。

go语言中的字符串是类型string。字符串的表示方法与其他编译型语言C/C++一样，用双引号包含，同时go又提供另外一种可以不转义的字符串表示方法：用反引号包含字符串时，其中的转衣符不会被转义(除了回车符)。

以下是几个例子：

``` go
var a int8 = 2
var b float32 = 0.1
var c string = "呵呵"
var d string = `\"\\`
```

注：go语言跟其它的C/C++/JAVA在变量声明和定义时有些不同：

* 可以使用类型推导，也就是可以省略其中的类型名
* 可以使用操作符`:=`直接定义变量并赋值

但是，建议在实际开发时还是尽量声明变量为变量设置明确的类型。

### 4 流程控制(条件if和switch、循环for)

if语句结构如下：

``` go
var a = 3

if a > 2 {
	fmt.Println("a > 2")
} else {
	fmt.Println(" a <= 2")
}
```

注意：

1 if判断的条件不需要括号
2 大括号的位置不是随意的，必须如上面所示的位置(不能像其他编译型语言一样可以随意放置)

switch语句结构如下：

``` go
var gender = "male"

switch gender {
case "male":
	fmt.Println("男人")
case "female":
	fmt.Println("女人")
default:
	fmt.Println("未知")
}
```

注意：语法上跟if类似，但是与其他语言有个区别是case结构里面不需要break

for语句有两种结构，一种是跟C/C++类似的结构，另一种是跟python类似的range结构(用于遍历字符串、数组、字典等可迭代的结构，一般来说，迭代时会返回两个元素，一个是元素的位置或者键，另一个是对应的值)：

``` go
// 结构一
for i := 0; i < 10; i++ {
    fmt.Println(i)
}

// 结构二
for k, v := range map[int]string{1:"a", 2:"b", 3:"c"} {
    fmt.Printf("%d => %s", k, v)
}
```

### 5 数组和对象(字典)

为了表示一系列相同类型的元素，数组也是一门语言不可缺少的类型。

go语言中定义数组的方式如下：`var a [3]int`，定义好后，它的内容会被初始化为对应类型的默认值。

与python类似，go语言也引入了切片类型，切片类型也通常是用于获取数组的一部分的最常用的方法:

``` go
var numbers3 = [5]int{1, 2, 3, 4, 5}
slice3 := numbers3[2 : len(numbers3)]
```

另一个比较常用的数据类型就是字典: `var a map[int]string`。

``` go
// 定义一个键类型为int，值类型为string的map
var a map[int]string = map[int]string{}

// 向字典中添加元素
a[1] = "test"

// 删除字典中的元素
delete(a, 1)

// 查询字典中的元素
// val返回查询到的值，ok返回键是否在字典中
val, ok := a[2]
```

### 6 函数

函数的传参规则跟C/C++一样：只有值传递，也就是说，传递时会复制实参给形参。为了可以修改实参，常见的方法就时传指针，另外传指针还有给好处：减少大对象的拷贝开销。

``` go
func my_func(para1 string, para2 string) string {
    return para1 + para2
}
```

### 7 输入输出

bufferio.NewWriter

### 8 异常处理

### 9 其它

除了以上的内容，go语言中还包含自身的一些特性：

* 结构体
* 接口
* 指针
* channel
* defer

#### 9.1 结构体

go中的结构体与其他的编译型语言类似，也有着封装属性和操作的功能。
另外，结构体有两个特点：结构体不是引用类型，是值类型；结构体不能继承。

第一步，声明结构体:

``` go
type Student struct {
    Name string
    Gender string
    Age int
}
```

第二步，必要的话，也可以为结构体增加函数:

``` go
func (st *Student) Grow() {
    st.Age++
}
```

第三步，定义结构体，并给结构体赋值:

``` go
var s Student = Student{Name: "jack", Gender: "male"}
```

第四步，可以对结构体调用相应的函数:

``` go
s.Grow()
```

#### 9.2 接口

同样的，与C/C++语言对比，与虚函数的概念类似，用于实现多态。

最常用的方式是：定义某类行为的接口，所有实现该接口的类型都可以调用该接口的方法，那么就可以对这些类型进行统一处理，从而实现多态。

另外，在golang中也有对于接口的一些特殊特性：

* 空接口`interface{}`可以表示任何类型(或者说任何类型都实现了空接口)，因此当函数的参数是个空接口，则可以传入任何参数，然后在函数体中对传入的参数进行类型判断，然后再执行不同的操作。

#### 9.3 指针

golang中的指针概念与C/C++一样，表示的也是地址，但是在使用上与C/C++有些区别：

* 指针也可以通过new()方法创建，不过不需要进行销毁，因为golang有垃圾回收

#### 9.4 channel

管道部分可以参看goroutine部分。

#### 9.5 defer

defer语句用于在退出函数时执行一定的动作，该动作通常时一些清理操作，类似于推出函数时自动调用某些对象的析构函数，跟python中引入的with类似。因此，谈到defer语句最常用的场景就时文件操作完毕后的关闭动作:

``` go
func readFile(path string) ([]byte, error) {
    file, err := os.Open(path)
    if err != nil {
        return nil, err
    }
    defer file.Close()
    return ioutil.ReadAll(file)
}
```

在函数执行结束时，无论是否发生异常，都会执行file.Close()

使用defer要注意三点：

* 有多个defer时，执行顺序跟代码顺序相反
* 执行到defer时，defer后的表达式并不会真正执行，只有当函数正常退出或者被剥夺控制权时才会执行，但是其中的表达式的参数却是在执行defer语句时就会求值
* defer语句后面接匿名函数时，要避免闭包常见的坑：当匿名函数使用defer语句外的变量时，需要将该变量作为参数，如果直接使用，得到的结果会并非预期
* 当执行defer语句时会将defer语句中的表达式记录下来，当异常发生时也会执行该函数，但是如果异常在defer语句之前发生，则不会执行defer中的语句。


go tool tour
http://docscn.studygolang.com/doc/
