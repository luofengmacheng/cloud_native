## 使用IDA查看二进制

### 1 IDA是什么？

[IDA](https://hex-rays.com/)是一款反编译软件，可以查看二进制的汇编代码，常用于逆向和问题定位。与其他商业软件类似，IDA也提供了很多类型的版本，基础的用法可以使用`IDA Free`。本文会简单介绍下IDA的基本用法。

### 2 IDA界面

![IDA主界面](https://github.com/luofengmacheng/cloud_native/blob/master/security/pics/ida_surface.jpg)

* 最上面是功能菜单和快捷按钮
* 在快捷按钮下方有彩色的条带色块，不同的颜色表示不同的段，例如，浅黄色是数据段，黄色是代码段，当选中某行代码时，就会在该位置显示小的绿色箭头
* 主界面的左侧是函数列表，右侧显示汇编代码
* 最下面就是输出窗口，例如对文件进行分析是否成功

### 3 汇编基础

按照汇编指令的风格，主要分成两种：

* Intel汇编：x86架构汇编语言中使用的风格，被Windows平台采用
* AT&T汇编：x64架构汇编语言中使用的风格，被Linux平台采用

两种汇编语言一个比较大区别就是操作数的顺序，Intel汇编的源操作数在右边，目的操作数在左边，而AT&T则刚好相反。

每条汇编指令包含两个部分：操作码和操作数，有的指令只有一个操作数，有的指令有多个操作数。

常用的操作码(不同的汇编的操作码的名称有小的差别)：

* move：将数据从一个位置移动到另一个位置，两个位置就是两个操作数
* load/store：将数据加载到寄存器/将数据从寄存器保存到内存
* cmp：比较两个操作数，并设置比较结果
* test：
* jmp/je/jz/jne/jnz/jg：跳转，可以直接跳转，也可以根据前一条的比较结果进行跳转
* call：函数调用
* ret：函数返回
* push/pop：将数据压栈/出栈

操作数的类型：

* 立即数：常量值
* 寄存器：如rax
* 内存地址：使用基址、变址、相对寻址的方式指定内存地址
* 标签：用于跳转类指令
* 字符串常量：用于表示代码中的字符串常量

### 4 IDA查看hello world二进制

![二进制的代码段起始部分](https://github.com/luofengmacheng/cloud_native/blob/master/security/pics/ida_start.jpg)

上述截图是代码段的起始部分，`_start`函数是一个特殊的函数，表示代码程序的入口点，在`_start`函数的最后调用了main函数：`call cs:__libc_start_main_ptr`，call指令用于调用函数，cs是代码段的基址寄存器，其中保存着代码段的基地址，__libc_start_main_ptr是一个特殊的符号，由libc定义，指向一个函数指针，该函数会执行一系列的初始化工作，然后再调用我们的main函数。

![IDA查看hello world二进制](https://github.com/luofengmacheng/cloud_native/blob/master/security/pics/ida_hello_world.jpg)

`main proc`和`main endp`用于定义main函数。

rbp和rsp结合起来实现对函数的调用，其中，rbp是Base Pointer寄存器，函数调用过程中保存的是帧的基地址。rsp是Stack Pointer寄存器，堆栈指针寄存器，始终指向当前线程的堆栈顶部。在函数内部的指令执行时，rbp不变，如果使用了函数级别的变量，则rsp会更新。

任何函数调用的开头两行指令通常是：

* push rbp：将当前的rbp保存到栈中，当前的rbp保存的是上一个函数的基地址
* mov rbp, rsp：将栈地址保存到帧寄存器

此时，rbp=rsp，而rbp+1(栈是向低地址增长的)的位置处保存的是上一个函数的基地址。

任何函数调用的结尾两行指令通常是：

* pop rbp：将栈顶的值保存到rbp，此处栈顶的值就是上一个函数的基地址
* retn：函数返回，与call指令相对应

去掉main函数中与函数调用相关的指令，剩下的4条指令为：

* lea rax, s：s是hello world字符串的地址，lea质量将字符串的地址放到rax寄存器中
* mov rdi, rax：将刚才得到的字符串的地址放到rdi中，该寄存器是函数调用的第一个参数
* call _puts：调用_puts函数输出字符串
* mov eax, 0：将eax寄存器设置为0，该寄存器作为函数的返回值，此处就是main函数的返回值

如果直接在linux中使用objdump查看二进制，看到的汇编代码如下：

![objdump查看hello world二进制](https://github.com/luofengmacheng/cloud_native/blob/master/security/pics/ida_objdump.jpg)

前面也说过，Linux采用的是AT&T格式的汇编，源操作数在右边，而windows上的则相反，另一个区别就是寄存器名称，AT&T的前面带有%，而Intel汇编则没有，除了这些，使用objdump得到的汇编代码和IDA一样。

### 5 查看带有条件语句和函数调用的二进制

在查看某个函数的调用关系时，按下`空格键`可以切换到图形显示方式。

![main函数](https://github.com/luofengmacheng/cloud_native/blob/master/security/pics/ida_func_main.jpg)

main函数跟hello world的差别不大，唯一就是在调用func1时，通过esi和edi传递了两个参数。

![函数调用](https://github.com/luofengmacheng/cloud_native/blob/master/security/pics/ida_func_call.jpg)

func1函数的开始部分就是获取两个参数，通过一系列mov操作，两个参数放到了帧的顶部和edx/eax中，然后调用_printf函数，使用cmp指令比较第一个参数和0，实用jnz判断比较的结果，左边的`红色`的分支表示jnz判断失败，右边`绿色`的分支表示jnz判断成功，因此，当第一个参数不为0时，就执行右边，当参数为0时，就执行左边，分支执行完成后，两个分支都会沿着`蓝色`的流程继续执行，调用func2，然后将func2的结果保存到第一个变量，然后函数结束。

IDA将整个执行流程以框图的形式展示，当其中有分支时，通过`绿色`和`红色`的分支线表示判断成功和失败的执行流，`蓝色`的线表示顺序的执行流。在了解这种用法后，遇到复杂的调用，可以直接跳过某些流程，直接定位到要查看的代码的附近。

### 6 总结

IDA是一款可以查看二进制的汇编级别代码的工具，并且提供了图形的方式展示程序的执行流程，有时候可以用来定位汇编级别的代码崩溃问题。
