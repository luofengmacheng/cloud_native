## 利用eBPF技术实现容器进程的阻断

### 1 题目

指定容器id和要阻断的进程名如whoami，运行demo后，在容器里面执行whoami，执行失败

### 2 eBPF介绍

BPF(伯克利包过滤)本身是一种包过滤机制，用来对网卡的流量进行旁路的截获，然后进行分析，主要应用场景为网络数据分析。后面出现了eBPF(extended BPF)，虽然它本身包含BPF，但是不论从技术还是应用场景上看，跟BPF差别较大，而并不是对BPF进行扩展，为了跟原来的BPF区分，将之前的BPF称为cBPF(classic BPF)，eBPF使得`软件定义内核`成为了可能。eBPF的主要应用场景有观测、网络、安全等。

### 3 eBPF的应用场景

#### 3.1 观测

* kprobes/kretprobes：内核中函数的动态跟踪，可以跟踪Linux内核中的函数入口或者返回点
* uprobes/uretprobes：用户级别的动态跟踪，跟踪的函数为用户程序中的函数
* tracepoints：内核静态跟踪
* perf_events：定时采样和PMC

#### 3.2 网络

* 包过滤
* 流量控制
* XDP(越早处理数据包，收益越大)：数据包转发、DDOS、分布式防火墙、ACL

#### 3.3 安全

* Seccomp
* KRSI

### 4 实现思路

* 阻断命令的执行：从上面eBPF的应用场景来看，可以使用kprobe跟踪execve系统调用，然后在bpf程序中获取execve的第一个参数(程序名称)，如果第一个参数与用户输入的命令一致，则进行阻断
* 对容器阻断：如果是使用runc等作为运行时，容器共用宿主机的内核，在容器中执行的系统调用也可以被宿主机上的程序跟踪和截获

### 5 使用eBPF阻断shell的调用

#### 5.1 eBPF hello world

学习一项新的开发技能，一般是从hello world开始的，以下就是ebpf的hello world程序，该程序会hook内核的execve调用，执行该程序后，当在系统中执行任何命令(执行命令其实就是用shell启动一个新的进程，然后调用execve)，都会打印Hello, BPF World!。

* 安装依赖：apt install build-essential git make libelf-dev clang strace tar bpfcc-tools linux-headers-$(uname -r) gcc-multilib
* 下载内核代码：[The Linux Kernel Archives](https://www.kernel.org/)，然后将内核代码拷贝到/kernel-src
* git clone https://github.com/bpftools/linux-observability-with-bpf
* cd linux-observability-with-bpf/code/chapter-2/hello_world/
* make bpfload
* ./monitor-exec

![eBPF Hello World](https://github.com/luofengmacheng/container_doc/blob/master/ebpf/pics/ebpf_hello_world.jpg)

该程序包含两个文件：

bpf_program.c就是内核需要加载的程序，用clang编译成字节码bpf_program.o后，再用loader.c进行加载。因此，loader.c中比较简单，就2个函数：load_bpf_file(加载bpf模块)，read_trace_pipe(打印trace信息)。重点就是bpf_program.c。

bpf_program.c中重点就是两个地方：字节码的挂载点和hook函数。

``` c
// 声明该函数的挂载点是execve系统调用
SEC("tracepoint/syscalls/sys_enter_execve")
int bpf_prog(void *ctx) {
  char msg[] = "Hello, BPF World!";

  // 简单的打印上面的输出即可
  bpf_trace_printk(msg, sizeof(msg));
  return 0;
}
```

#### 5.2 eBPF获取系统调用参数

从上面可以看到，eBPF程序主要就是编写bpf模块的程序，但是里面既涉及到一些非常规函数，而且还涉及到用户态和内核态，学习成本太高。好在现在有BCC这个工具可以方便开发。

BCC是一个可以帮助快速开发eBPF程序的工具，语言可以绑定到python和Lua。例如，上面的hello world程序也可以写成：

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

上面的syscall__execve()函数已经有文件名参数，因此filename就是命令的路径，现在有了命令的路径，剩下的就是：

* 如果命令跟要阻断的命令一致，则执行阻断逻辑
* 如何阻断呢？bpf提供对返回值的重写：bpf_override_return();，其中第一个参数就是syscall__execve()的第一个参数，第二个参数就是errno定义的值，这里设置为-1，含义是：Operation not permitted

至此，就可以对命令进行阻断了。

### 6 阻断containerd容器中的命令

#### 6.1 安装containerd

* 下载containerd：wget https://github.com/containerd/containerd/releases/download/v1.6.1/cri-containerd-cni-1.6.1-linux-amd64.tar.gz
* 安装：tar -C / cri-containerd-cni-1.6.1-linux-amd64.tar.gz && containerd config default > /etc/containerd/config.toml
* 启动：wget -o /usr/lib/systemd/system/containerd.service https://raw.githubusercontent.com/containerd/containerd/main/containerd.service && systemctl start containerd

* 拉取镜像：ctr i pull uhub.service.ucloud.cn/edn_docker/nginx:1.20
* 创建容器：ctr run -d -t uhub.service.ucloud.cn/edn_docker/nginx:1.20 nginx
* 进入容器：ctr t exec -exec-id $RANDOM -t nginx sh

#### 6.2 阻断容器

进入容器，然后执行命令可以发现，宿主机上面的程序同样可以捕获execve系统调用，但是这样做存在的问题是阻断了所有的命令调用。因此，现在就剩下一个问题：在执行阻断逻辑时需要识别出当前所处的容器。

通过对以bpf_为开头的函数进行查看，发现里面有个函数：bpf_get_current_cgroup_id()，用于获取cgroup的id，而容器之间、容器与宿主机之间就处于不同的cgroup，是否能够以该值进行区分呢？

在syscall__execve()函数中打印该值，然后进行测试发现：

* 宿主机的该值与容器的该值不同
* 无论何时，只要容器还活着，该值就不会变化(如果重启是会变化的)

说明，可以在syscall__execve()函数中对cgroup_id进行区分是否处于容器中，因此，还有一个问题，当输入容器ID时如何能够获取到该值呢？暂时没有找到方法，因此，这里的办法是：

* 先执行一个eBPF程序，然后进入给定的容器中执行一个预设的命令，在输出中找到cgroup_id
* 将cgroup_id作为变量注入到python的eBPF程序中

这样做就可以对容器中的命令进行阻断，但是这里有2个变量：

* 容器
* 命令

这里使用类似模版的方式，在python脚本中program是一个模版，里面有2个变量：容器的cgroup_id和命令，当获取到参数后对program进行替换就可以得到最终的bpf模块。

最终的效果就是：

![eBPF阻断容器中的命令](https://github.com/luofengmacheng/container_doc/blob/master/ebpf/pics/ebpf_interrupt_containerd_container.jpeg)

### 7 参考文档

* [一文看懂eBPF｜eBPF的简单使用](https://mp.weixin.qq.com/s/V-5k1mX5JRA0lWLXJ2AxpA)
* [云原生安全攻防｜使用eBPF逃逸容器技术分析与实践](https://security.tencent.com/index.php/blog/msg/206)
* [bpf: implement bpf_get_current_cgroup_id() helper](https://patchwork.ozlabs.org/project/netdev/patch/20180603225943.2370719-2-yhs@fb.com/)