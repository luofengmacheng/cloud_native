## C++中的全局变量以及static和extern关键字

### 1 声明和定义

声明就是告诉编译器有这个东西的存在，而定义则是这个东西的实现。

对于变量来说，声明就是告诉编译器存在这个名称的变量，定义则是给这个变量分配内存并赋值：

``` C++
// 变量声明，声明时不能赋值，如果进行赋值，就是定义
extern int var;

// 变量定义，定义时可以为其赋值，并且此时赋值是个好的习惯
int var = 0;
```

对于函数来说，声明就是告诉编译器存在这个名称的函数，定义则是这个函数的实现。

函数的声明就是给出函数的返回值、函数名和参数类型：

``` C++
// 函数声明
int add(int a, int b);

// 函数定义
int add(int a, int b) {
	return a + b;
}
```

声明和定义的区分主要用于全局变量，毕竟，局部变量不需要区分声明和定义。需要记住的是：全局范围内，变量的声明可以有多个，而定义只能有一个。

### 2 static

被static修饰的全局变量称为静态全局变量，静态全局变量的作用域是当前文件，也就是说，不能使用extern关键字将该变量导入到其他文件访问。

如下示例：

``` C++
// module.h
#ifndef LUO
#define LUO

static int var;

#endif

// module.cpp
#include <iostream>
#include "module.h"

void func() {
	var = 2;
	printf("var=%d address=%p\n", var, &var);
}

// main.cpp
#include <iostream>
#include "module.h"

extern void func();

int main() {
	func();
	printf("var=%d address=%p\n", var, &var);
}
```

将全局变量放到头文件中，然后在两个文件中使用，执行时可以发现，两个变量的地址不一样，也就是说，虽然这个变量在两个文件中，但是他们其实是不同的变量。

总之，对于static的全局变量，需要记住：它们只能用在当前文件，尽量不要放在头文件中（因为头文件大概率是要被多个源文件引用的）。

static不仅可以修饰全局变量，还可以修饰局部变量，当修饰局部变量时，就修改了变量的声明周期，它就不是存储在栈上，而是存储在全局数据区。

``` C++
#include <iostream>

void func() {
        static int a = 0;

        ++a;
        printf("%d\n", a);
}

int main() {
        func();
        func();
}
```

这里将func()函数中的变量a用static修饰，执行时会发现，当下一次再次执行时a就是上次执行的值。这样的变量通常可以用于只在某个函数中使用全局变量，也就是要求它的声明周期是全局的，但是使用范围却是某个函数中。

对于函数而言，用static修饰，表明该函数只在当前文件中使用。

### 3 extern

前面已经说过，extern通常用来声明变量和函数，表明变量在其他地方定义，此处只是告诉编译器有这个东西而已。

因此，extern比较常用的方式就是在头文件中声明变量和函数：

``` C++
// module.h
#ifndef LUO
#define LUO

extern int var;
extern void func();

#endif

// module.cpp
#include <iostream>
#include "module.h"

int var = 0;

void func() {
	var = 2;
	printf("var=%d address=%p\n", var, &var);
}

// main.cpp
#include <iostream>
#include "module.h"

int main() {
	func();
	printf("var=%d address=%p\n", var, &var);
}
```

在头文件module.h中声明变量和函数，然后在module.cpp中定义变量和函数，最后在main.cpp中引入头文件，就可以在main.cpp中使用变量和函数了。这种方式就是extern的常规用法。

当然，对于这里的例子，还可以直接将extern的变量和函数放到main.cpp中，由链接器在链接阶段去查找：

``` C++
#include <iostream>

extern void func();
extern int var;

int main() {
        func();
        printf("var=%d address=%p\n", var, &var);
}
```

extern的另一个用法就是链接C语言库。