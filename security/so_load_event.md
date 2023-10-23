## 获取so加载事件

### 0 前言

在某些场景下，我们需要获取so加载事件

### 1 动态库的编译和使用

so动态库通常用于编写插件，将插件的逻辑写在so中，然后主程序只是负责调用so中的插件的入口函数。

使用so的方式1：隐式调用，直接在编译时指定要用的动态库

``` c
// libplugin.c so中的函数的具体实现代码
#include <stdio.h>

void plugin_func() {
    printf("plugin func called");
}
```

``` c
// main.c 主程序，负责调用so中的函数
#include <stdio.h>

int main() {
    plugin_func();
}
```

编译和运行：

``` shell
# 先将libplugin.c编译为libplugin.so
gcc -fPIC -shared libplugin.c -o libplugin.so

# 编译主程序
gcc -pie -fPIC -L. main.c -lplugin -ldl -o main

# 运行主程序
LD_LIBRARY_PATH=. ./main
```

LD_LIBRARY_PATH环境变量：程序执行时会查找需要的so，默认会在/lib和/usr/lib下查找，因此，如果需要的so不在这两个路径中，就需要设置LD_LIBRARY_PATH环境变量，程序执行时就会去设置的路径中查找。

使用so的方式2：显式调用，使用dlopen和dlsym显示加载和调用

``` c
// main.c
#include <stdio.h>
#include <dlfcn.h>

typedef void (*plugin_func_t)();

int main() {
    void *handle = dlopen("./libplugin.so", RTLD_LAZY);
    if (handle == NULL) {
        printf("dlopen failed: %s\n", dlerror());
        return 1;
    }

    plugin_func_t func = dlsym(handle, "plugin_func");
    if (func == NULL) {
        printf("plugin_func failed\n");
        return 1;
    }

    func();

     if(dlclose(handle) != 0) {
        printf("dlclose failed: %s\n", dlerror());
        return 1;
    }

    return 0;
}
```

编译和运行：

``` shell
# 编译主程序
gcc -fPIC main.c -ldl -o main

# 运行主程序
./main
```

用dlopen打开要加载的so文件，此时才会将so读入内存，然后用dlsym获取so中的函数的地址后执行。

### 2 可能的思路

* LD_PRELOAD：[Linux hook: Ring3下动态连接库.so函数劫持](https://www.cnblogs.com/reuodut/articles/13723437.html)，编写自己的dlopen()函数去覆盖ld.so中的dlopen()，但是这种方案只能用于先执行自己的程序加载自己的dlopen()函数后才行
* ptrace：ptrace可以拦截系统调用，而dlopen中肯定会有打开文件的操作，那是不是通过拦截open或者openat系统调用，然后查看参数来实现呢，这种方式比较麻烦，而且ptrace的性能比较差
* uprobe：对dlopen函数进行插桩，获取dlopen调用的事件，但是如果不是通过dlopen来加载so呢
* ebpf/uprobe：ebpf对uprobe进行了支持，但是ebpf有内核版本的问题

### 3 uprobe

uprobe是一种用户态的插装技术，想象一种场景，需要统计某个函数的运行次数，对于开发人员，可以在函数中增加计数逻辑，那对于测试或者其他人员呢？如果现在要统计另一个函数呢？不可能每个函数都去增加这个逻辑。这时，可以对这个函数进行插桩，当系统调用到这个函数时可以进行统计计数，不需要计数了，移除该桩就行。

#### 3.1 uprobe的简单用例

在使用uprobe进行插桩之前，需要明确要插装的函数在二进制中的地址，将该地址告诉uprobe，uprobe才能在调用到该函数时执行额外的逻辑。

![使用objdump获取函数在二进制中的偏移量](https://github.com/luofengmacheng/cloud_native/blob/master/security/pics/objdump_libdl_dlopen.jpeg)

可以看到dlopen在libdl.so中的偏移量是：0000000000001010。

然后可以配置uprobe：

``` shell
# 配置uprobe探测点
echo 'p:dlopen /lib64/libdl-2.17.so:0x0000000000001010 +0(%di):string' > /sys/kernel/debug/tracing/uprobe_events

# 开启监控
echo 1 > /sys/kernel/debug/tracing/events/uprobes/enable
```

然后就可以在/sys/kernel/debug/tracing/trace中查看收到的调用记录：

![使用uprobe探测调用dlopen的结果](https://github.com/luofengmacheng/cloud_native/blob/master/security/pics/uprobe_dlopen_result.jpeg)

#### 3.2 uprobe的语法

uprobe_tracer的整体格式为：

```
p[:[GRP/]EVENT] PATH:OFFSET [FETCHARGS] : 设置入口探测点
r[:[GRP/]EVENT] PATH:OFFSET [FETCHARGS] : 设置返回探测点
p[:[GRP/]EVENT] PATH:OFFSET%return [FETCHARGS] : 设置返回探测点
-:[GRP/]EVENT                           : 删除探测点
```

* p和r前缀表示probe和retprobe，分别设置入口和出口的探测点
* GRP：探测点的分组，默认是uprobes
* EVENT：给探测点设定的名称
* PATH：二进制的路径
* OFFSET：设定探测点的偏移量
* OFFSET%return：设置返回探测点的偏移量
* FETCHARGS：
	* %REG：获取REG寄存器的值
	* @ADDR：获取ADDR地址的内存
	* @+OFFSET：获取二进制文件的偏移量地址的值
	* $stackN：获取栈的第N个
	* $stack：获取栈的地址
	* $retval：获取返回值，只能用于retprobe
	* $comm：获取当前的task comm
	* +|-[u]OFFS：
	* \IMM：
	* NAME=FETCHARG：
	* FETCHARG:TYPE：

#### 3.3 uprobe原理

当定义uprobe时，内核会在探测点用断点指令(int 3)替换，当程序执行到该指令时，内核将触发事件，程序陷入到内核态，并以回调函数的方式调用探针函数，执行完探针函数再返回到用户态继续执行后续的指令。

因此，当开启监控后，就相当于在指定的地址的位置加上调用探针函数的逻辑。

#### 3.4 使用uprobe时遇到的问题

fault：

![uprobe获取参数时显示fault](https://github.com/luofengmacheng/cloud_native/blob/master/security/pics/uprobe_result_fault.jpeg)

unknown type：

![uprobe直接输出unknown type](https://github.com/luofengmacheng/cloud_native/blob/master/security/pics/uprobe_result_unknown_type.jpeg)

### 5 要解决的问题

#### 5.1 隐式调用

对dlopen进行插桩后发现，/sys/kernel/debug/tracing/trace文件中不是整个程序运行时所有的so都能够看到，因为这里只是对dlopen进行了插桩，因此，如果是隐式调用so的方式则无法采集到。

[Linux加载启动可执行程序的过程(一)内核空间加载ELF的过程](https://blog.csdn.net/chrisnotfound/article/details/80082289)

[Linux加载启动可执行程序的过程(二)解释器完成动态链接](https://blog.csdn.net/chrisnotfound/article/details/80082463)

也就是说，隐式调用的so是在二进制运行过程中由动态链接器负责加载的，动态链接器调用open和mmap将so文件映射到内存，因此，如果想要获取隐式调用so的事件，初步的想法是通过系统调用，但是像open和mmap这类系统调用又比较常规，这种采集方式相当于获取了所有的调用，然后再从中过滤，对性能就有较大影响。

通过对ld加载so的流程进行分析，显示调用和隐式调用so的调用栈如下：

![显示调用和隐式调用的调用栈](https://github.com/luofengmacheng/cloud_native/blob/master/security/pics/so_load_stack.jpg)

会发现，如果可以跟踪_dl_map_object的调用，可以获取两种方式的so的加载，对_dl_map_object进行跟踪：

![跟踪_dl_map_object](https://github.com/luofengmacheng/cloud_native/blob/master/security/pics/trace_dl_map_object.jpg)

但是这里也可以发现，这里只有so的名称，但是没有具体路径，出现多次应该是在系统中搜索对应的so。

#### 5.2 偏移量的获取

使用uprobe时需要知道dlopen在libdl.so中的偏移量，或者直接使用符号名。

如果直接使用符号名，需要考虑符号名不存在或者变更的场景。

#### 5.3 流式读取

配置好uprobe后，就需要读取结果，如果是人读取可以直接读取/sys/kernel/debug/tracing/trace文件，如果是程序呢？

在trace文件相同的目录下有trace_pipe可以提供不断的数据流，可以用于程序的读取。

#### 5.4 整体流程

* 线程1：
    * 获取当前进程加载的libdl.so
    * 解析libdl.so，获取dlopen的偏移量
    * 生成uprobe_trace，并启用
* 线程2：
    * 流式读取/sys/kernel/debug/tracing/trace_pipe
    * 生成so加载事件

### 6 潜在的风险

#### 6.2 操作系统兼容性（操作系统类型和内核版本）

测试过程中发现，ubuntu 21.10中进程没有加载libdl.so，具体原因未知。同时，ubuntu中使用objdump命令查看libdls.so的符号表时没有找到symtab节，因此，拿不到dlopen的偏移量。

在centos 7.5(3.10.0-862)中大量进程的dlopen调用的参数都是打印(fault)，而在ubuntu 21.10(5.13.0-52)上面也有少量的(fault)。

当使用[perf-tools](https://github.com/brendangregg/perf-tools)中的uprobe工具时会有对内核版本的提示：

![perf-tools中uprobe对内核版本的提示](https://github.com/luofengmacheng/cloud_native/blob/master/security/pics/uprobe_kernel_version.jpg)

#### 6.1 uprobe自身机制的稳定性

#### 6.3 对系统的影响

uprobe提供的debugfs接口进行操作，如果系统中已经存在其他的uprobe_tracer，那么只需要增加探测点即可，并且需要保证名称不重复。

#### 6.3 对性能的影响

uprobe是通过替换对应地址的指令然后陷入内核的方式实现，如果不是调用频率特别高的调用，对系统基本没有影响。

### 7 结论

* 需要明确uprobe使用过程中的问题的原因(fault、unknown type)
* 需要明确uprobe在操作系统发行版和内核的边界
* 需要确定隐式so调用的采集方式

### 参考文档

* [Linux原生跟踪工具Ftrace必知必会](https://bbs.csdn.net/topics/608810664)
* [ebpf user-space probes原理探究](https://www.edony.ink/deep-in-ebpf-uprobe/)
* [使用uprobe跟踪分析GO函数的参数](https://www.bilibili.com/read/cv19487471/)
* [ld.linux.so源码分析--dl_main](https://blog.csdn.net/yejing_utopia/article/details/45025685)