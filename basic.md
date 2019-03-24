## go语言基础

### 1 前言

一门编程语言的基础知识一般包括以下几个部分：

* Hello World
* 基本数据类型
* 运算符和表达式
* 流程控制(选择if、循环for)
* 数组和对象(字典)
* 函数
* 输入输出

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

### 6 数组和对象(字典)

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

### 7 函数

``` go
func my_func(para1 string, para2 string) string {
    return para1 + para2
}
```

### 8 其它

除了以上的内容，go语言中还包含自身的一些特性：

* channel
* 结构体、接口、指针
* 

#### 8.1 channel

#### 8.2 结构体

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