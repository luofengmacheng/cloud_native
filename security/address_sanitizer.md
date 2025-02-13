## 使用AddressSanitizer分析内存非法使用问题

### 1 为什么需要AddressSanitizer？

Valgrind是比较常用的内存问题定位工具，既然已经有了Valgrind，为什么还需要AddressSanitizer呢？

与Valgrind相比，AddressSanitizer存在以下优势：

* Valgrind通过模拟CPU来检测内存错误，导致会以较慢的速度运行程序；而AddressSanitizer是在编译阶段插入检查的逻辑，执行速度比Valgrind快很多
* Valgrind是一个独立的工具，可以使用在任何程序上；而AddressSanitizer与编译器紧密集成，可以在构建时自动启用
* 在错误信息的展示上，AddressSanitizer提供的错误信息比Valgrind容易理解
* AddressSanitizer作为编译器的一部分，通过编译选项启用；而Valgrind作为独立的工具，需要更多的配置和学习才能使用
* AddressSanitizer通过编译时插桩和运行时检查来检测内存错误，误报率较低

从使用场景来说，AddressSanitizer专注于发现`内存未释放`和`访问非法内存`的问题。

### 2 如何使用AddressSanitizer

从gcc 4.8开始，AddressSanitizer称为gcc的一部分，但是，gcc 4.8的AddressSanitizer没有符号信息，建议使用gcc 4.9及以上版本。

首先使用AddressSanitizer来检测下常见的内存问题。

用例一：未正确释放内存

``` C++
#include <iostream>

int main() {
	int *ptr = new(int);
	*ptr = 0;

	std::cout << *ptr << std::endl;
}
```

上述代码申请了一个int的空间，但是没有释放，编译该程序：`g++ -fsanitize=address -fno-omit-frame-pointer -o main main.cpp`，然后直接执行：

```
0

=================================================================
==711759==ERROR: LeakSanitizer: detected memory leaks

Direct leak of 4 byte(s) in 1 object(s) allocated from:
    #0 0x7f37f92201c7 in operator new(unsigned long) ../../../../src/libsanitizer/asan/asan_new_delete.cpp:99
    #1 0x55581dc742be in main (/root/asan/main+0x12be)
    #2 0x7f37f8c54fcf in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:58

SUMMARY: AddressSanitizer: 4 byte(s) leaked in 1 allocation(s).
```

结果比Valgrind更好理解：

* 出现了内存泄漏
* 导致内存泄漏的内存分配的调用栈
* 总结，在一次分配过程中泄漏了4字节

用例二：分配的内存未使用配对的函数释放

``` C++
#include <iostream>

int main() {
	int *ptr = new(int);
	*ptr = 0;

	free(ptr);
}
```

```
=================================================================
==711960==ERROR: AddressSanitizer: alloc-dealloc-mismatch (operator new vs free) on 0x602000000010
    #0 0x7f8ef531c517 in __interceptor_free ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:127
    #1 0x5571e62742ef in main (/root/asan/main+0x12ef)
    #2 0x7f8ef4d52fcf in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:58
    #3 0x7f8ef4d5307c in __libc_start_main_impl ../csu/libc-start.c:409
    #4 0x5571e62741c4 in _start (/root/asan/main+0x11c4)

0x602000000010 is located 0 bytes inside of 4-byte region [0x602000000010,0x602000000014)
allocated by thread T0 here:
    #0 0x7f8ef531e1c7 in operator new(unsigned long) ../../../../src/libsanitizer/asan/asan_new_delete.cpp:99
    #1 0x5571e627429e in main (/root/asan/main+0x129e)
    #2 0x7f8ef4d52fcf in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:58

SUMMARY: AddressSanitizer: alloc-dealloc-mismatch ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:127 in __interceptor_free
==711960==HINT: if you don't care about these errors you may set ASAN_OPTIONS=alloc_dealloc_mismatch=0
==711960==ABORTING
```

结果也可以直接看出来：

* alloc-dealloc-mismatch(operator new vs free)：分配和销毁不匹配
* 分配和销毁的调用栈

用例三：使用已经被回收的内存

``` C++
#include <iostream>

int main() {
	int *ptr = new(int);
	*ptr = 0;

	delete(ptr);
	*ptr = 1;
}
```

```
=================================================================
==712018==ERROR: AddressSanitizer: heap-use-after-free on address 0x602000000010 at pc 0x560c4aac2331 bp 0x7ffd02a2d040 sp 0x7ffd02a2d030
WRITE of size 4 at 0x602000000010 thread T0
    #0 0x560c4aac2330 in main (/root/asan/main+0x1330)
    #1 0x7ff2fbc22fcf in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:58
    #2 0x7ff2fbc2307c in __libc_start_main_impl ../csu/libc-start.c:409
    #3 0x560c4aac21c4 in _start (/root/asan/main+0x11c4)

0x602000000010 is located 0 bytes inside of 4-byte region [0x602000000010,0x602000000014)
freed by thread T0 here:
    #0 0x7ff2fc1ef22f in operator delete(void*, unsigned long) ../../../../src/libsanitizer/asan/asan_new_delete.cpp:172
    #1 0x560c4aac22f9 in main (/root/asan/main+0x12f9)
    #2 0x7ff2fbc22fcf in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:58

previously allocated by thread T0 here:
    #0 0x7ff2fc1ee1c7 in operator new(unsigned long) ../../../../src/libsanitizer/asan/asan_new_delete.cpp:99
    #1 0x560c4aac229e in main (/root/asan/main+0x129e)
    #2 0x7ff2fbc22fcf in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:58

SUMMARY: AddressSanitizer: heap-use-after-free (/root/asan/main+0x1330) in main
Shadow bytes around the buggy address:
  0x0c047fff7fb0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x0c047fff7fc0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x0c047fff7fd0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x0c047fff7fe0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x0c047fff7ff0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
=>0x0c047fff8000: fa fa[fd]fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c047fff8010: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c047fff8020: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c047fff8030: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c047fff8040: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c047fff8050: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07 
  Heap left redzone:       fa
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
  Left alloca redzone:     ca
  Right alloca redzone:    cb
  Shadow gap:              cc
==712018==ABORTING
```

会输出heap-use-after-free类型的结果：

* 结果包含内存分配、释放以及写已经释放的内存的调用栈
* 在SUMMARY下面出现两块数据，这个就涉及到AddressSanitizer的实现

### 3 AddressSanitizer的原理

前面说过，AddressSanitizer主要的使用场景是`内存未释放`和`访问非法内存`，为了能够发现这两种情况，需要为内存保存一些状态信息。

为了能够为内存保存状态信息，AddressSanitizer会用自己的运行时库替换默认的malloc/free，将需要操作内存的代码进行替换。

例如，如果是读操作：

``` C++
... = *address;
```

则在访问内存前执行检查：

``` C++
if (IsPoisoned(address)) {
    ReportError(address, kAccessSize, kIsWrite);
}
... = *address;
```

如果是写操作：

``` C++
*address = ...;
```

则在写操作之前进行检查：

``` C++
if (IsPoisoned(address)) {
    ReportError(address, kAccessSize, kIsWrite);
}
*address = ...;
```

其中的`IsPoisoned()`函数就是用来检查内存地址是否`合法`。

AddressSanitizer使用`影子内存`的机制进行内存地址的合法性检查：对程序实际使用的内存使用额外的字节来存储它的状态信息。

使用1个字节存储实际使用的8个字节的状态：

* 8个字节的数据可读写，则1个字节的值为0
* 8个字节的数据不可读写，则1个字节的值为负数，不同的值表示不同类型的内存，如0xfa表示堆左边的redzone(redzone是在正常可以使用的内存两侧的边界)，0xf1表示栈左边的redzone
* 8个字节中前k个字节可读写，后8-k个字节不可读写，则1个字节的值为k

因此，`IsPoisoned()`函数就是检查地址对应的影子内存中的状态是否可以访问，如果不能访问，则出现`访问非法内存`的问题，如果访问的地址的影子内存是redzone，则说明出现`内存访问越界`的问题。

基于上述描述，再来看上面`heap-use-after-free`的输出的后面部分的内容：

```
Shadow bytes around the buggy address:
  0x0c047fff7fb0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x0c047fff7fc0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x0c047fff7fd0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x0c047fff7fe0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x0c047fff7ff0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
=>0x0c047fff8000: fa fa[fd]fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c047fff8010: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c047fff8020: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c047fff8030: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c047fff8040: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c047fff8050: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07 
  Heap left redzone:       fa
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
  Left alloca redzone:     ca
  Right alloca redzone:    cb
  Shadow gap:              cc
==712018==ABORTING
```

第二部分是图例，也就是影子内存中不同的值代表的含义，例如，fd表示已经被释放的内存。

第一部分显示的就是影子内存的数据，出现了fd表示已经被释放的内存，对该内存再次访问就会出现heap-use-after-free问题。

下面再看一个栈溢出的例子：

``` C++
#include <iostream>
#include <string.h>

int main() {
	int arr1[10];
	int arr2[12];

	memcpy(arr1, arr2, 12*sizeof(int));
}
```

结果：

```
=================================================================
==714984==ERROR: AddressSanitizer: stack-buffer-overflow on address 0x7ffe915be528 at pc 0x7f22667182c3 bp 0x7ffe915be4d0 sp 0x7ffe915bdc78
WRITE of size 48 at 0x7ffe915be528 thread T0
    #0 0x7f22667182c2 in __interceptor_memcpy ../../../../src/libsanitizer/sanitizer_common/sanitizer_common_interceptors.inc:827
    #1 0x5620a92e0349 in main (/root/asan/main+0x1349)
    #2 0x7f22661c8fcf in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:58
    #3 0x7f22661c907c in __libc_start_main_impl ../csu/libc-start.c:409
    #4 0x5620a92e01c4 in _start (/root/asan/main+0x11c4)

Address 0x7ffe915be528 is located in stack of thread T0 at offset 72 in frame
    #0 0x5620a92e0298 in main (/root/asan/main+0x1298)

  This frame has 2 object(s):
    [32, 72) 'arr1' (line 5)
    [112, 160) 'arr2' (line 6) <== Memory access at offset 72 partially underflows this variable
HINT: this may be a false positive if your program uses some custom stack unwind mechanism, swapcontext or vfork
      (longjmp and C++ exceptions *are* supported)
SUMMARY: AddressSanitizer: stack-buffer-overflow ../../../../src/libsanitizer/sanitizer_common/sanitizer_common_interceptors.inc:827 in __interceptor_memcpy
Shadow bytes around the buggy address:
  0x1000522afc50: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1000522afc60: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1000522afc70: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1000522afc80: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1000522afc90: 00 00 00 00 00 00 00 00 00 00 00 00 f1 f1 f1 f1
=>0x1000522afca0: 00 00 00 00 00[f2]f2 f2 f2 f2 00 00 00 00 00 00
  0x1000522afcb0: f3 f3 f3 f3 00 00 00 00 00 00 00 00 00 00 00 00
  0x1000522afcc0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1000522afcd0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1000522afce0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1000522afcf0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07 
  Heap left redzone:       fa
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
  Left alloca redzone:     ca
  Right alloca redzone:    cb
  Shadow gap:              cc
==714984==ABORTING
```

结果表明，这是一个stack-buffer-overflow的错误，对于栈内存来说，也会给出分配栈空间的变量的地方以及对栈内存出现非法访问的调用栈，还分别给出了两个变量的位置。

下面的影子内存可以看出：

* f1是栈左红区，f2是栈中红区，f3是栈右红区
* 所有的栈空间都处于f1和f3之间，变量之间用f2隔开
* arr1是10个int，也就是40个字节，使用5个影子字节，即影子内存中`=>`所在行的左侧的5个字节
* arr2是12个int，也就是48个字节，使用6个影子字节，即影子内存中`=>`所在行的左侧的6个字节
* `[f2]`表明访问栈中红区，出现栈的访问越界

### 4 总结

AddressSanitizer是进行内存异常使用分析的工具，该工具已经集成到编译器中，因此，只能用于分析C/C++语言的内存问题分析。与Valgrind相比，运行速度更快，但是，从场景来说，AddressSanitizer主要用于检测`内存的非法使用`，当然也包括`内存未正确释放`的问题，而Valgrind则可以分析出导致内存增长的调用栈。

因此：

* 如果出现内存偏高的问题，可以使用Valgrind工具分析
* 如果出现内存导致的core问题，可以使用gdb的watchpoint或者AddressSanitizer分析
