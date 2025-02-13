## eBPF编程指南(三)：用户态和内核态的数据交互

### 1 背景

使用eBPF时通常需要从内核态获取数据，然后将数据传递到用户态进行后续处理，因此，如何完成用户态和内核态之间的数据交互是eBPF程序中非常重要的部分，而且，不同的场景需要使用不同的方式。

### 2 调试工具：bpf_trace_printk和bpf_printk

eBPF程序是不太好调试的，也没有类似gdb的工具，而且，eBPF程序一个典型的场景是用于数据采集，在数据量比较大的时候，就算有gdb之类的工具也不好使用。

在开发阶段和问题可复现的场景，经常使用输出语句进行调试，例如，C语言中的`printf`和Java语言中的`System.out.println`，而在eBPF中则有`bpf_trace_printk`用于输出。

在内核代码的`include/uapi/linux/bpf.h`中有对`bpf_trace_printk`函数的原型说明和描述：

```
long bpf_trace_printk(const char *fmt, u32 fmt_size, ...)
```

`bpf_trace_printk`函数的第一个参数是要输出的字符串，第二个是字符串的长度，`const char *fmt`可以是个格式化字符串，可以使用常见的`%d`、`%s`、`%x`等格式化字符，后面就是用于填充格式化字符串的具体值，使用`bpf_trace_printk`时需要注意的是：

* `bpf_trace_printk`最多只能有5个参数，因此，格式化字符串中只能有3个格式化字符
* 输出的内容可以在`/sys/kernel/debug/tracing/trace`或者`/sys/kernel/debug/tracing/trace_pipe`中查看
* `bpf_trace_printk`非常慢，只能用于调试，不能用于生产环境

``` C
SEC("tracepoint/syscalls/sys_enter_execve")
int tracepoint_execve(struct syscall_trace_enter *ctx) {
    u64 id = bpf_get_current_pid_tgid();
    pid_t pid = (pid_t)id;

    char pname[16];
    bpf_get_current_comm(&pname, sizeof(pname));

    char msg[] = "new process: pid=%d pname=%s";
    bpf_trace_printk(msg, sizeof(msg), pid, pname);

    return 0;
}
```

上述代码会在执行execve系统调用时输出一行信息，在其中打印执行execve的pid和进程名。之后，当每次执行命令都可以在trace文件中查看到相关的输出。

为了更加方便地使用`bpf_trace_`

### 3 perf_events


### 4 通过ring buffer传递



### 5 通过map传递

### 6 参考文档

* [（译）BPF技巧和窍门：bpf_trace_printk() 和 bpf_printk() 指南](https://blog.csdn.net/q547569552/article/details/119919114)

