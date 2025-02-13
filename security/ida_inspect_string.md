## IDA查看std::string的各种操作

### 1 前言

相对C语言来说，C++中增加了很多特性，例如，构造函数、析构函数、异常处理等，因此，从汇编级别的语言看，C++就复杂很多。下面使用IDA工具查看std::string的一些操作背后的实现。

以下程序在`Ubuntu 21.10(5.13.0-52-generic)`编译，g++版本为`gcc version 11.2.0 (Ubuntu 11.2.0-7ubuntu2)`。

### 2 创建std::string

``` C++
#include <iostream>

int main() {

	std::string str("abc");

	return 0;
}
```

![](https://github.com/luofengmacheng/cloud_native/blob/master/security/pics/ida2_string1.jpg)

![](https://github.com/luofengmacheng/cloud_native/blob/master/security/pics/ida2_string2.jpg)

图中灰色的部分是调用的相关函数，基本就是函数的原型。

* 调用std::allocator<char>::allocator()
* 将unk_2008的字符串（也就是给定的初始化字符串abc）地址给到rsi，将allocator()的返回值给到rdi，然后调用std::string的初始化函数
* 然后分别调用allocator和std::string的析构函数，说明此时main函数已经执行结束，剩下的指令可以看做是main函数退出时执行的清理操作

程序结束时的清理逻辑包含两个部分：

* 如果程序抛出异常，则会执行异常处理函数，此处的异常处理函数中调用了allocator的析构函数
* 检查栈是否会溢出，如果是，则调用`__stack_chk_fail`进行检查

以下代码在分析时会忽略清理逻辑，直接查看程序主体逻辑。

### 3 std::string.find()

``` C++
#include <iostream>

int main() {

	std::string str("abc");
	auto iter = str.find('a');
	if(iter == std::string::npos) {
		std::cout << "not found" << std::endl;
	}

	return 0;
}
```

![](https://github.com/luofengmacheng/cloud_native/blob/master/security/pics/ida2_string3.jpg)

* 调用std::string.find()函数
* 将返回值保存到临时变量，然后将该临时变量与全1比较，如果不为0判断成功，则直接结束，如果不为0判断失败，也就是为0，也就是相等，则执行中间的输出逻辑
* 输出时，先将字符串地址通过rax中转后放到rsi，再将cout函数的地址放到rdi，然后创建ostream对象并调用ostream对象的operator<<函数进行输出

### 4 std::string.substr()

``` C++
#include <iostream>
#include <string>

int main() {

	std::string str("abc");
	size_t pos = str.find('a');
	std::cout << str.substr(pos-2) << std::endl;

	return 0;
}
```

![](https://github.com/luofengmacheng/cloud_native/blob/master/security/pics/ida2_string4.jpg)

需要注意的是，如果直接使用`g++ -o main main.cpp`编译上述程序，在程序崩溃后，使用gdb查看core文件时，无法查看C++库的堆栈，如果要想看到上述的堆栈，需要使用静态编译：`g++ -o main -static main.cpp`。

![](https://github.com/luofengmacheng/cloud_native/blob/master/security/pics/ida2_string5.jpg)

执行substr()时，先比较第一个参数和字符串长度，然后通过比较的结果进行跳转，ja指令表示大于则跳转，因此，这里的含义是，如果第一个参数大于字符串长度，则跳转到右侧的框内，然后组装好异常描述信息后抛出异常。因此，在执行substr()时需要先比较起始索引位置和字符串长度。

### 5 std::string.replace()

``` C++
#include <iostream>
#include <string>

int main() {

	std::string str("abc");
	size_t pos = str.find('a');
	str.replace(pos, 2, "dd");
	std::cout << str << std::endl;

	return 0;
}
```

这里的调用跟上一个用例类似，先调用std::string的初始化函数，然后调用find()和replace()函数。

不过在使用replace()函数时，可能会出现下面这个堆栈：

![](https://github.com/luofengmacheng/cloud_native/blob/master/security/pics/ida2_string6.jpg)

这是因为，在调用std::string::replace()函数时，如果发现字符串的长度有变化，就会重新申请空间。

![](https://github.com/luofengmacheng/cloud_native/blob/master/security/pics/ida2_string7.jpg)

上面的汇编代码可以看出，首先调用_malloc函数分配内存，如果分配失败，则调用std::get_new_handler()，如果调用成功，则执行新std::get_new_handler()返回的函数，然后再次调用_malloc分配内存，如果还失败，则结束，如果调用std::get_new_handler()返回为空，则抛出异常退出。

因此，走到上述堆栈需要满足两个条件：

* 调用_malloc分配内存失败
* 没有设置new_handler，new_handler是在执行operator new失败时的处理函数，可以在里面销毁部分内存

### 6 总结

std::string提供了更高层次的字符串操作，并且提供了很多的方法，与C风格字符串相比，使用起来更加方便，但是，在这些方法的底层，编译器默默做了很多事情。当程序编译为二进制后，可以在Linux平台使用objdump工具查看汇编级别的代码，使用IDA则可以很直观的看到各操作之间的调用关系，因此，如果是定位core问题，或者想看某个操作具体做了什么，IDA工具是不二之选。当然，由于指令比较多，这里没有详细介绍每行指令的含义，但是，通过部分指令后面的函数名以及跳转关系，基本可以熟悉整体的调用流程。

本文使用IDA工具查看了std::string的创建、find()、substr()、replace()函数的流程。
