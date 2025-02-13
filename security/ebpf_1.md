## eBPF编程指南(一)：eBPF初体验

### 1 什么是EBPF？

EBPF是一种可以让程序员在内核态执行自己的程序的机制，但是，为了安全起见，无法像内核模块一样随意调用内核的函数，只能调用一些bpf提前定义好的函数。为了让内核执行程序员自己的代码，需要指定HOOK点，相当于当内核执行到某个地方时就会执行程序员的代码，这有点类似于ftrace。

EBPF程序包含两个部分：

* 内核态代码：编译为.o文件，在内核态执行，将数据发送到队列
* 用户态代码：将内核态代码的.o文件加载到内核，接收队列中的数据并分析

### 2 Hello World(BCC)

EBPF有多种开发方式，按照语言来分的话，有两种方式：一种是内核态代码和用户态代码都用C语言开发；另一种是内核态代码用C语言开发，而用户态代码可以其他语言开发，例如，Python、Golang、Lua等。

[BCC](https://github.com/iovisor/bcc)是一个方便构建EBPF程序的工具，让编写EBPF程序更加简单，内核态代码依旧采用C语言，而用户态代码可以采用Python和Lua。

下面就是一个简单的Hello world版本的EBPF程序，该程序对execve系统调用进行HOOK，内核态程序只执行一行打印语句，用户态程序负责将数据打印出来，因此，每当系统中创建一个进程，该程序就会输出一行Hello, BPF World!。

在执行之前，需要根据操作系统安装一些包：[Installing BCC](https://github.com/iovisor/bcc/blob/master/INSTALL.md)。

``` python
from bcc import BPF

program = """
#include <uapi/linux/ptrace.h>

int syscall__execve(struct pt_regs *ctx,
    const char __user *filename,
    const char __user *const __user *__argv,
    const char __user *const __user *__envp)
{
    char msg[] = "Hello, BPF World!";

    bpf_trace_printk("%s", msg);
    return 0;
}
"""

b = BPF(text=program)
execve_fnname = b.get_syscall_fnname("execve")
b.attach_kprobe(event=execve_fnname, fn_name="syscall__execve")
b.trace_print()
```

可以看到，使用BCC开发EBPF程序还是比较简单的，用户态程序只需要定义好要HOOK的点，并接收数据，主要的逻辑其实还是在内核态代码中。

但是，这种方式不便于理解EBPF的工作方式，比如，program这段程序代码是怎么载入到内核的呢？载入之前需要编译吗？那Python里面是怎么编译的呢？

### 3 EBPF的基于C语言的开发模式

要理解EBPF的工作方式，最好的方式还是基于纯C语言开发EBPF程序。

如果用C语言开发，内核态代码和用户态代码就必须分开写，内核态代码就需要知道可以调用哪些函数，而用户态代码就需要知道如何载入内核态编译后的二进制。

现在常用的开发方式有两种，一种是基于原生libbpf库开发，另一种是基于libbpf-bootstrap开发。

### 4 基于原生libbpf库

首先是环境的搭建：

* 安装依赖：yum install clang elfutils-libelf-devel，其中clang用于编译内核态代码，elfutils-libelf-devel用于处理内核态代码编译成的二进制
* 安装libbpf：git clone https://github.com/libbpf/libbpf && cd libbpf/src && make && make install
* 生成vmlinux.h，解除对kernel header的依赖：bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h

内核态代码和用户态代码都需要引入的头文件：

``` C
#ifndef __EXECVE_H
#define __EXECVE_H

struct event {
    pid_t pid;
    pid_t ppid;
};

#endif
```

内核态代码：

``` C
#include "vmlinux.h"
#include "execve.h"
#include <bpf/bpf_helpers.h>

static const struct event empty_event = {};

struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 1024);
    __type(key, pid_t);
    __type(value, struct event);
} execs SEC(".maps");

struct {
    __uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
    __uint(key_size, sizeof(u32));
    __uint(value_size, sizeof(u32));
} events SEC(".maps");

SEC("tracepoint/syscalls/sys_enter_execve")
int tracepoint_execve(struct syscall_trace_enter *ctx) {
    u64 id = bpf_get_current_pid_tgid();
    pid_t pid = (pid_t)id;
    pid_t tgid = id >> 32;

    if(bpf_map_update_elem(&execs, &pid, &empty_event, BPF_NOEXIST)) {
        return 0;
    }

    struct event *event = bpf_map_lookup_elem(&execs, &pid);
    if(event == NULL) {
        return 0;
    }

    event->pid = tgid;

    bpf_perf_event_output(ctx, &events, BPF_F_CURRENT_CPU, event, sizeof(struct event));

    bpf_map_delete_elem(&execs, &pid);
    return 0;
}

char _license[] SEC("license") = "GPL";
```

用户态代码：

``` C
#include <stdio.h>
#include <errno.h>
#include <bpf/libbpf.h>
#include <bpf/bpf.h>
#include "execve.h"

#define PERF_BUFFER_PAGES 64
#define PERF_POLL_TIMEOUT_MS 100

static void handle_event(void *ctx, int cpu, void *data, __u32 data_sz) {
    const struct event *e = data;

    printf("pid:%d\n", e->pid);
}

int main() {
    int err;
    // 打开内核程序生成的二进制文件
    struct bpf_object *bpf_obj = bpf_object__open("execve_kern.o");
    if(bpf_obj == NULL) {
        printf("open .o faile failed\n");
        return -1;
    }

    // 载入内核程序
    int ret = bpf_object__load(bpf_obj);
    if(ret != 0) {
        printf("load bpf object failed\n");
        return -1;
    }

    // 根据内核函数的函数名找到内核程序
    struct bpf_program *bpf_prog = bpf_object__find_program_by_name(bpf_obj, "tracepoint_execve");
    struct bpf_link *bpf_link = bpf_program__attach(bpf_prog);
    if(libbpf_get_error(bpf_link)) {
        return -1;
    }

    // 找到事件的map
    int fd = bpf_object__find_map_fd_by_name(bpf_obj, "events");

    // 根据事件的map创建perf_buffer，并指定perf_buffer中的数据的回调处理函数handle_event
    struct perf_buffer *pb = NULL;
    pb = perf_buffer__new(fd, PERF_BUFFER_PAGES, handle_event, NULL, NULL, NULL);
    if(pb == NULL) {
        printf("open perf buffer failed\n");
        goto cleanup;
    }

    while(true) {
        err = perf_buffer__poll(pb, PERF_POLL_TIMEOUT_MS);
        if(err < 0 && err != -EINTR) {
            printf("poll failed\n");
            goto cleanup;
        }
        err = 0;
    }

cleanup:
    perf_buffer__free(pb);
    bpf_link__destroy(bpf_link);
    bpf_object__close(bpf_obj);

    return 0;
}
```

* 编译内核态程序：clang -g -target bpf -D__TARGET_ARCH_x86 -c execve_kern.c -o execve_kern.o
* 编译用户态程序：gcc -g -Wall -c execve_user.c -o execve_user.o
* 将用户态程序编译为完整的二进制：gcc -g -Wall execve_user.o -o execve -lbpf

执行`./execve`时，如果有新的进程就会打印进程的Pid。

### 5 基于libbpf-bootstrap脚手架

基于libbpf的方式有个缺点：最终的二进制包含内核态程序编译的二进制execve_kern.o和用户态程序编译的二进制execve，不便于分发。

第二种方式也是基于libbpf库，只是在上层做了一些封装，并且可以整体打成一个二进制。

``` shell
git clone https://github.com/libbpf/libbpf-bootstrap

# 下载依赖的仓库的代码
cd libbpf-bootstrap && git submodule update --init --recursive
```

libbpf-bootstrap仓库本身的内容只有examples(示例代码)，以及与vmlinux.h相关的tools和vmlinux目录，主要依赖其他的仓库：

* blazesym：地址符号化和反向查找
* bpftool：用于生成skel文件和vmlinux.h文件
* libbpf：也就是上面的libbpf库

examples/c中的minimal.bpf.c和minimal.c就是最简单的示例代码。

``` C
// minimal.bpf.c
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

char LICENSE[] SEC("license") = "Dual BSD/GPL";

int my_pid = 0;

SEC("tp/syscalls/sys_enter_write")
int handle_tp(void *ctx)
{
	int pid = bpf_get_current_pid_tgid() >> 32;

	if (pid != my_pid)
		return 0;

	bpf_printk("BPF triggered from PID %d.\n", pid);

	return 0;
}
```

内核态代码hook write系统调用，当用户态程序执行write时，就打印一行输出，这里的my_pid会在用户态程序中被替换。

``` C
#include <stdio.h>
#include <unistd.h>
#include <sys/resource.h>
#include <bpf/libbpf.h>
#include "minimal.skel.h"

static int libbpf_print_fn(enum libbpf_print_level level, const char *format, va_list args)
{
	return vfprintf(stderr, format, args);
}

int main(int argc, char **argv)
{
	struct minimal_bpf *skel;
	int err;

	/* 安装libbpf库的错误和调试函数 */
	libbpf_set_print(libbpf_print_fn);

	/* 打开内核态程序 */
	skel = minimal_bpf__open();
	if (!skel) {
		fprintf(stderr, "Failed to open BPF skeleton\n");
		return 1;
	}

	/* 将当前进程的pid更新到内核态程序中 */
	skel->bss->my_pid = getpid();

	/* 加载和验证内核态程序 */
	err = minimal_bpf__load(skel);
	if (err) {
		fprintf(stderr, "Failed to load and verify BPF skeleton\n");
		goto cleanup;
	}

	/* 将内核态程序附着到挂载点 */
	err = minimal_bpf__attach(skel);
	if (err) {
		fprintf(stderr, "Failed to attach BPF skeleton\n");
		goto cleanup;
	}

    // 打印一行提示，可以通过查看/sys/kernel/debug/tracing/trace_pipe文件内容查看内核程序的输出
	printf("Successfully started! Please run `sudo cat /sys/kernel/debug/tracing/trace_pipe` "
	       "to see output of the BPF programs.\n");

	for (;;) {
		/* 执行下面的fprintf会调用write系统调用，就会触发内核态挂载点函数的执行 */
		fprintf(stderr, ".");
		sleep(1);
	}

cleanup:
	minimal_bpf__destroy(skel);
	return -err;
}
```

对上述代码进行编译：

``` shell
make minimal
```

会在当前路径下生成minimal二进制，执行minimal就可以看到效果，此时，内核态二进制代码也在minimal二进制中。

### 6 libbpf-boostrap中示例的编译过程

在执行`make minimal`时会看到整体的编译过程：

* 创建本地的编译目录：.output
* 编译libbpf：编译libbpf-bootstrap/libbpf，并将中间文件和最终生成.a文件放到.output/libbpf中
* 编译bpftool：编译libbpf-bootstrap/bpftool
* 将minimal.bpf.c编译为minimal.bpf.o
* 使用bpftool工具将minimal.bpf.o转换成minimal.skel.h
* 将minimal.c编译为minimal.o
* 生成最终的二进制minimal

但是，上述命令只能看到大概的编译过程，如果想看到执行编译的完整命令，可以执行`make minimal -e V=1`。

例如，编译过程中会打印以下信息：

```
...                        libbfd: [ OFF ]
...               clang-bpf-co-re: [ on  ]
...                          llvm: [ OFF ]
...                        libcap: [ OFF ]
```

其实是因为在编译bpftool过程中对上述库进行分别测试。

最终编译minimal的代码使用了以下命令：

``` shell
# 使用clang将minimal.bpf.c编译为minimal.tmp.bpf.o
clang -g -O2 -target bpf -D__TARGET_ARCH_x86		      \
	     -I.output -I../../libbpf/include/uapi -I../../vmlinux/x86/ -I/root/ebpf/libbpf-bootstrap/blazesym/capi/include -idirafter /usr/bin/../lib/clang/18/include -idirafter /usr/local/include -idirafter /usr/include		      \
	     -c minimal.bpf.c -o .output/minimal.tmp.bpf.o

# 使用bpftool将minimal.bpf.o编译为minimal.tmp.bpf.o
/root/ebpf/libbpf-bootstrap/examples/c/.output/bpftool/bootstrap/bpftool gen object .output/minimal.bpf.o .output/minimal.tmp.bpf.o

# 使用bpftool将minimal.bpf.o转换成minimal.skel.h
/root/ebpf/libbpf-bootstrap/examples/c/.output/bpftool/bootstrap/bpftool gen skeleton .output/minimal.bpf.o > .output/minimal.skel.h

# 使用gcc将minimal.c编译为minimal.o
cc -g -Wall -I.output -I../../libbpf/include/uapi -I../../vmlinux/x86/ -I/root/ebpf/libbpf-bootstrap/blazesym/capi/include -c minimal.c -o .output/minimal.o

# 使用gcc将minimal.o编译为minimal
cc -g -Wall .output/minimal.o /root/ebpf/libbpf-bootstrap/examples/c/.output/libbpf.a   -lelf -lz -o minimal
```

在最终生成minimal二进制的过程中，主要依赖三个库：libbpf、libelf、libzlib，其中，只有libbpf是静态库，另外两个都是动态库。

### 7 参考文档

* [perf-tools](https://github.com/brendangregg/perf-tools)：基于ftrace和perf开发的性能分析工具
* [libbpf-tools](https://github.com/iovisor/bcc/tree/master/libbpf-tools)：基于libbpf开发的工具
* [BPF Performance Tools](https://github.com/brendangregg/bpf-perf-tools-book)：`BPF性能之巅`的配套代码
* [eBPF动手实践系列三：基于原生libbpf库的eBPF编程改进方案](https://blog.csdn.net/weixin_48534929/article/details/136816342)
* [libbpf-bootstrap](https://github.com/libbpf/libbpf-bootstrap)：libbpf的脚手架项目
* [BPF Documentation](https://www.kernel.org/doc/html/latest/bpf/)：Linux内核中的BPF文档
* [BPF Portability and CO-RE](https://facebookmicrosites.github.io/bpf/blog/2020/02/19/bpf-portability-and-co-re.html)
* [elfutils](https://sourceware.org/elfutils/)