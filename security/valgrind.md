## 使用Valgrind定位内存增长问题

### 1 内存问题

内存问题是一类比较难以定位的问题，通常有两类场景：

* 程序在`低负载情况`下的内存使用量是否正常，低负载情况下不应该太高
* 程序在`高负载情况`下的内存使用量是否正常，高负载情况下应该为内存使用量设置阈值，防止触发OOM

因此，为了检查内存使用量是否正常，需要知道`当前的内存使用量`以及`哪里分配的内存造成内存增长`。

使用`top`和`ps`命令可以查看某个进程的内存使用量，然后结合代码中初始分配的内存大小评估当前的内存使用量是否正常。

### 2 Valgrind

#### 2.1 Valgrind介绍

Valgrind是Linux上的一套开源的动态分析工具集，通常用来检测和分析程序中的错误，提高程序的稳定性和性能。

Valgrind整体架构上包含内核和周边工具集，将程序放到内核模拟的仿真环境中运行，并提供一些能力接口，然后基于这些接口实现周边工具。

#### 2.2 Valgrind中的Memcheck

Memcheck是最常用且是默认的工具，通常用于检查内存泄漏等问题。

示例1：

``` C++
#include <iostream>

int main() {
	int *ptr = nullptr;
	*ptr = 0;

	return 0;
}
```

编译并执行：

```
g++ -o mem3 -g mem3.cpp
valgrind --leak-check=full ./mem3
```

输出：

```
==43290== Memcheck, a memory error detector
==43290== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==43290== Using Valgrind-3.17.0 and LibVEX; rerun with -h for copyright info
==43290== Command: ./mem3
==43290== 
==43290== Invalid write of size 4
==43290==    at 0x10917D: main (mem3.cpp:5)
==43290==  Address 0x0 is not stack'd, malloc'd or (recently) free'd
==43290== 
==43290== 
==43290== Process terminating with default action of signal 11 (SIGSEGV)
==43290==  Access not within mapped region at address 0x0
==43290==    at 0x10917D: main (mem3.cpp:5)
==43290==  If you believe this happened as a result of a stack
==43290==  overflow in your program's main thread (unlikely but
==43290==  possible), you can try to increase the size of the
==43290==  main thread stack using the --main-stacksize= flag.
==43290==  The main thread stack size used in this run was 8388608.
==43290== 
==43290== HEAP SUMMARY:
==43290==     in use at exit: 0 bytes in 0 blocks
==43290==   total heap usage: 1 allocs, 1 frees, 72,704 bytes allocated
==43290== 
==43290== All heap blocks were freed -- no leaks are possible
==43290== 
==43290== For lists of detected and suppressed errors, rerun with: -s
==43290== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
Segmentation fault
```

从上面的输出可以看出：

* mem3.cpp的第5行进行了地址的非法写操作
* 程序收到SIGSEGV信号而终止，是由于访问了0x0地址
* 所有的堆内存都被释放，没有内存泄漏

示例2：

``` C++
#include <iostream>

int main() {
	int *ptr = (int*)malloc(sizeof(int));
	*ptr = 0;

	return 0;
}
```

输出：

```
==45775== Memcheck, a memory error detector
==45775== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==45775== Using Valgrind-3.17.0 and LibVEX; rerun with -h for copyright info
==45775== Command: ./mem4
==45775== 
==45775== 
==45775== HEAP SUMMARY:
==45775==     in use at exit: 4 bytes in 1 blocks
==45775==   total heap usage: 2 allocs, 1 frees, 72,708 bytes allocated
==45775== 
==45775== 4 bytes in 1 blocks are definitely lost in loss record 1 of 1
==45775==    at 0x4843839: malloc (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==45775==    by 0x10919E: main (mem4.cpp:4)
==45775== 
==45775== LEAK SUMMARY:
==45775==    definitely lost: 4 bytes in 1 blocks
==45775==    indirectly lost: 0 bytes in 0 blocks
==45775==      possibly lost: 0 bytes in 0 blocks
==45775==    still reachable: 0 bytes in 0 blocks
==45775==         suppressed: 0 bytes in 0 blocks
==45775== 
==45775== For lists of detected and suppressed errors, rerun with: -s
==45775== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
```

从上面的输出可以看出：

* `in use at exit`表明有4个字节在程序退出时还在使用，下面的`4 bytes in 1 blocks are definitely lost`也表明有内存没有释放
* `LEAK SUMMARY`是对内存泄漏的汇总信息

Memcheck可以检测程序中的常见的内存分配但不释放、访问越界等问题，但是，有时候有指针指向内存块，只是程序还没有释放且继续申请内存，运行一段时间后由于OOM退出，这时候Memcheck就无能为力了，而且现代的程序比较复杂，Memcheck会有很多误告。

#### 2.3 Valgrind中的Massif

内存持续增长一般有两种场景：

* 出现死循环或者递归不终止，造成栈溢出
* 持续分配堆内存，而不释放，此时可能存在内存泄漏或者是程序逻辑处理异常而没有正确释放内存(当然，也有可能是死循环引起的)

对于第一个问题，可以通过日志和进程当前的快照确认，而且，出现这种情况时通常会造成线程的CPU高。

对于第二个问题，就是要知道程序在哪里出现了持续分配堆内存，然后可以审查这部分代码，看是否存在没有释放的场景。那怎么知道程序在哪里分配堆内存呢？就要用到Valgrind中的Massif工具。

Massif工具通过采样的方式获取当前内存分配的情况，并且可以输出分配内存的调用栈。

示例代码：

``` C++
#include <iostream>
#include <unistd.h>

void func() {
	while(true) {
		char *ptr = (char *)malloc(1024*sizeof(char));
		sleep(1);
	}
}

int main() {
	func();

	return 0;
}
```

使用Valgrind执行该程序：`valgrind --tool=massif --time-unit=B ./mem5`，然后将输出文件转换成文本：`ms_print massif.out.47145 > memory.log`。

结果文件分为三个部分。

第一部分是一个简单的说明，表明生成该采样文件执行的命令、Massif和ms_print的参数。

由于Massif工具的工作方式是对堆内存的使用情况进行快照，因此，第二部分的坐标图和第三部分的表格记录的都是快照的情况，有的快照包含内存的使用量和分配堆内存的调用栈，有的快照只包含内存的使用量，因此将快照分为`详细快照`和`普通快照`。

第二个部分是一个用符号生成的坐标图：

```
    KB
122.4^                                             ##########################
     |                                           :@#
     |                                         :::@#
     |                                      ::@:::@#
     |                                    @:::@:::@#
     |                                  ::@:::@:::@#
     |                                :@::@:::@:::@#
     |                             ::::@::@:::@:::@#
     |                           ::@:::@::@:::@:::@#
     |                          :::@:::@::@:::@:::@#
     |                          :::@:::@::@:::@:::@#
     |                          :::@:::@::@:::@:::@#
     |                          :::@:::@::@:::@:::@#                         :
     |                          :::@:::@::@:::@:::@#                         :
     |                          :::@:::@::@:::@:::@#                         :
     |                          :::@:::@::@:::@:::@#                         :
     |                          :::@:::@::@:::@:::@#                         :
     |                          :::@:::@::@:::@:::@#                         :
     |                          :::@:::@::@:::@:::@#                         :
     |                          :::@:::@::@:::@:::@#                         :
   0 +----------------------------------------------------------------------->KB
     0                                                                   193.4

Number of snapshots: 55
 Detailed snapshots: [9, 19, 29, 39, 49, 53 (peak)]
```

坐标图的横轴表示采样的间隔，不同的单位有不同的含义：

* i：默认的单位，表示每次快照之间的指令执行数量
* ms：毫秒，表示从程序启动之后的时间
* B：字节，表示分配的内存量，单位是B

坐标图的纵坐标表示分配的堆内存大小，所以从图里面可以直观的看出堆内存分配的趋势。

坐标图中的符号的含义：

* 冒号：该字符组成的柱状图表示`普通快照`，只记录内存的使用量
* @：该字符组成的柱状图表示`详细快照`，记录内存的使用量和分配内存的调用栈
* 井号：该字符组成的柱状图表示`峰值快照`，记录内存的使用量和分配内存的调用栈

在坐标图的下面还列出了记录快照的数量：

* 总共记录了55个快照
* `详细快照`有6个，里面也包含`峰值快照`

因此，看第二个部分就可以看到大概的内存增长趋势：堆内存是一直增长的。

第三部分是记录的所有快照的数据，以前10个快照为例：

```
--------------------------------------------------------------------------------
  n        time(B)         total(B)   useful-heap(B) extra-heap(B)    stacks(B)
--------------------------------------------------------------------------------
  0              0                0                0             0            0
  1         72,712           72,712           72,704             8            0
  2         73,744           73,744           73,728            16            0
  3         74,776           74,776           74,752            24            0
  4         75,808           75,808           75,776            32            0
  5         76,840           76,840           76,800            40            0
  6         77,872           77,872           77,824            48            0
  7         78,904           78,904           78,848            56            0
  8         79,936           79,936           79,872            64            0
  9         80,968           80,968           80,896            72            0
99.91% (80,896B) (heap allocation functions) malloc/new/new[], --alloc-fns, etc.
->89.79% (72,704B) 0x4903B79: ??? (in /usr/lib/x86_64-linux-gnu/libstdc++.so.6.0.29)
| ->89.79% (72,704B) 0x4011B1D: call_init.part.0 (dl-init.c:70)
|   ->89.79% (72,704B) 0x4011C07: call_init (dl-init.c:33)
|     ->89.79% (72,704B) 0x4011C07: _dl_init (dl-init.c:117)
|       ->89.79% (72,704B) 0x4001109: ??? (in /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2)
|
->10.12% (8,192B) 0x1091BE: func() (mem5.cpp:6)
  ->10.12% (8,192B) 0x1091DB: main (mem5.cpp:12)
```

快照的信息组成一个表格：

* n：快照的序号，从0开始，对应的就是第二部分中详细快照中的序号
* time：时间，对应的是坐标图中的横轴，这里的time_unit设置的是B，因此，横轴的单位是B，而且由于横坐标是B，横纵坐标值是一样的
* total：分配的内存总量，对应的是坐标图中的纵轴，单位是B
* useful-heap：程序申请的内存量
* extra-heap：超过程序申请的内存量，例如管理内存需要的元数据，一般来说，这部分在整体分配的内存中的占比可以忽略
* stacks：栈的大小，默认情况下，栈的分析是关闭的，会降低Massif的速度，可以通过`--stacks=yes`开启

`普通快照`和`详细快照`都会包含上述信息，而`详细快照`还包含分配这些内存的调用栈。

例如，序号为9的快照就是`详细快照`，首先给出了程序申请的内存(useful-heap)占内存总量(total)的占比以及堆分配函数(malloc/new/new[])：

```
99.91% (80,896B) (heap allocation functions) malloc/new/new[], --alloc-fns, etc.
```

然后就是分配内存的堆栈，这里包含两个栈，第一个栈是：

```
->89.79% (72,704B) 0x4903B79: ??? (in /usr/lib/x86_64-linux-gnu/libstdc++.so.6.0.29)
| ->89.79% (72,704B) 0x4011B1D: call_init.part.0 (dl-init.c:70)
|   ->89.79% (72,704B) 0x4011C07: call_init (dl-init.c:33)
|     ->89.79% (72,704B) 0x4011C07: _dl_init (dl-init.c:117)
|       ->89.79% (72,704B) 0x4001109: ??? (in /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2)
```

该堆栈是Valgrind自身分配的内存，一般可以忽略。

第二个栈是：

```
->10.12% (8,192B) 0x1091BE: func() (mem5.cpp:6)
  ->10.12% (8,192B) 0x1091DB: main (mem5.cpp:12)
```

从该调用栈就可以知道8192B是从哪里分配的，注意调用栈是下面调用上面，也就是说，mem5.cpp的第6行的func()函数中的代码分配的。

有时候在调用栈中还会看到`02.18% (1,891,902B) in 546 places, all below massif's threshold (1.00%)`这样的描述，是因为有些小的内存分配对整体影响不大就没有记录调用栈，一般都可以直接忽略，如果想看到所有的调用栈，可以设置`--threshold`参数。

### 3 总结

当程序运行过程中，内存持续增长，一段时间后就可能造成OOM，此时可以使用Valgrind的Massif工具分析程序执行过程中的堆分配内存量。

在使用Valgrind的Massif工具运行程序后就可以得到进程在运行过程中的堆内存分配情况以及对应的栈，对分配内存较多的代码进行审查并优化，或者添加一些限制，减少分配的内存。

使用过程中需要注意两个地方：

* Massif工具只能用来分析`堆内存`的使用，也就是通过`malloc/new/new[]`分配的内存，有些可能是通过`mmap`分配的内存就没办法分析了
* 对于运行时间较短的程序，需要加上`--time-unit=B`参数
