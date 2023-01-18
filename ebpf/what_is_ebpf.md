## 什么是eBPF(Helloworld)

### 1 eBPF

eBPF提供了一种安全的方式可以在内核态执行代码，相比于用户态，拥有较高的性能，相比于内核模块，更加安全和简单，因此，被广泛用于可观测性、网络和安全等领域。

### 2 执行cilium/ebpf中的示例

编写ebpf程序通常需要两部分逻辑，一部分是内核态的ebpf程序，这部分通常用c语言编写，然后编译为.o文件，另一部分是用户态的程序，用户态的程序负责将ebpf程序加载到内核执行。

如果用golang编写用户态程序，常用的库有cilium的ebpf和libbpf-go，这里使用cilium的ebpf。

* 安装编译工具：yum install -y clang llvm
* 设置环境变量：export BPF_CLANG=clang
* 下载项目：git clone https://github.com/cilium/ebpf.git
* 编译：cd ebpf/examples/kprobe && go generate && go build
* 执行：./kprobe
* 打开其他窗口执行命令，就可以看到程序每秒打印一次，打印出统计的系统调用次数

### 3 构建自己的ebpf项目

* 创建自己的项目目录：mkdir hell_ebpf
* 拷贝ebpf程序：cp ebpf/examples/kprobe/kprobe.c hello_ebpf/
* 拷贝用户态的go程序：cp ebpf/examples/kprobe/main.go hello_ebpf/
* 拷贝头文件：cp -rf ebpf/examples/headers hello_ebpf/
* 修改hello_ebpf中的main.go中go:generate行，将最后面的headers目录从上个目录改成本目录
* go mod init hello_ebpf
* go mod tidy
* go generate
* go build
* ./hello_ebpf

### 4 示例中的kprobe程序解析

示例中的kprobe程序包含两个文件：一个是ebpf程序kprobe.c，另一个是用户态的go程序main.go。

kprobe.c包含两个部分：

``` c
// 定义一个map
struct bpf_map_def SEC("maps") kprobe_map = {
        .type        = BPF_MAP_TYPE_ARRAY,
        .key_size    = sizeof(u32),
        .value_size  = sizeof(u64),
        .max_entries = 1,
};

// 拦截内核的sys_execve系统调用
SEC("kprobe/sys_execve")
int kprobe_execve() {
        u32 key     = 0;
        u64 initval = 1, *valp;

        // 从map中查找key对应的值，这里key不会变，总是等于0
        // 如果没有获取到，则用初始的1更新
        valp = bpf_map_lookup_elem(&kprobe_map, &key);
        if (!valp) {
                bpf_map_update_elem(&kprobe_map, &key, &initval, BPF_ANY);
                return 0;
        }

        // 获取获取到，则自增
        __sync_fetch_and_add(valp, 1);

        return 0;
}
```

main.go：

``` golang
        fn := "sys_execve"

        // 将编译的bpf程序和maps加载到内核
        objs := bpfObjects{}
        if err := loadBpfObjects(&objs, nil); err != nil {
                log.Fatalf("loading objects: %v", err)
        }
        defer objs.Close()

        // 将函数与我们的ebpf程序关联
        // KprobeExecve通过golang中的标签与kprobe.c中的kprobe_execve关联
        kp, err := link.Kprobe(fn, objs.KprobeExecve, nil)
        if err != nil {
                log.Fatalf("opening kprobe: %s", err)
        }
        defer kp.Close()

        // Read loop reporting the total amount of times the kernel
        // function was entered, once per second.
        ticker := time.NewTicker(1 * time.Second)
        defer ticker.Stop()

        log.Println("Waiting for events..")

        for range ticker.C {
                var value uint64

                // 通过mapKey从map中获取值，然后打印
                // 这里的KprobeMap通过golang中的标签与kprobe.c中的kprobe_map关联
                if err := objs.KprobeMap.Lookup(mapKey, &value); err != nil {
                        log.Fatalf("reading map: %v", err)
                }
                log.Printf("%s called %d times\n", fn, value)
        }

```

### 5 编写ebpf程序的方式

上面提过，epbf一般要分为两个程序：一个是加载到内核的程序，这部分一般是用C语言编写，另一个是运行在用户态的程序，当前这部分可以绑定到不同的语言，常用的有python和golang。

#### 5.1 系统调用

[利用eBPF技术实现容器进程的阻断](https://github.com/luofengmacheng/cloud_native/blob/master/ebpf/ebpf_interrupt_container_process.md)

#### 5.2 libbpf(BTF & CO-RE)

* 安装依赖：yum install clang binutils-devel elfutils-libelf-devel
* 安装libbpf：git clone https://github.com/libbpf/libbpf && cd libbpf/src && make && make install
* 生成vmlinux.h，解除对kernel header的依赖：* bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h
* 编译内核代码：clang -g -O2 -target bpf -D__TARGET_ARCH_x86 -c kprobe.bpf.c -o kprobe.bpf.o
* 生成用户态头文件：bpftool gen skeleton kprobe.bpf.o > kprobe.skel.h
* 将用户态程序编译为目标文件：clang -g -O2 -Wall -c kprobe.c -o kprobe.o
* 将目标文件编译为可执行程序：clang -Wall -O2 -g kprobe.o -static -lbpf -lelf -lz -o kprobe -v(此处如果编译为静态版本，需要有[glibc](https://www.gnu.org/software/libc/)、[libelf](https://sourceware.org/elfutils/)、[zlib](https://www.zlib.net/)的静态链接包，如果没有就需要下载对应的源代码进行编译)

#### 5.3 BCC(python)

* yum install python3-bcc(会安装libbpf、python3-bcc、bcc等包)

``` python
from bcc import BPF

BPF(text='int kprobe__sys_clone(void *ctx) { bpf_trace_printk("Hello, World!\\n"); return 0; }').trace_print()
```

#### 5.4 cilium的ebpf(golang)

[什么是eBPF(Helloworld)](https://github.com/luofengmacheng/cloud_native/blob/master/ebpf/what_is_ebpf.md)

#### 5.6 上述方式的优劣

* 系统调用：使用系统调用的方式需要对
* libbpf(BTF & CO-RE)：
* BCC(python)：用户态程序编写方便，但是需要安装python和对应的库
* cilium的ebpf：

### 6 版本特性

* eBPF支持：3.15引入
* BTF支持：4.18

### 7 文档

* [Introduction to eBPF](https://houmin.cc/posts/2c811c2c/)
* [eBPF Map操作](https://houmin.cc/posts/98a3c8ff/)
* [eBPF, part 1: Past, Present, and Future](https://www.ferrisellis.com/content/ebpf_past_present_future/)
* [eBPF, part 2: Syscall and Map Types](https://www.ferrisellis.com/content/ebpf_syscall_and_maps/)
* [bcc Reference Guide](https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md)
* [centos 8 安装glibc-static、cannot find -lc 解决办法](https://blog.csdn.net/ab411919134/article/details/115079449)