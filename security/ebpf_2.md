## eBPF编程指南(二)：CO-RE

### 1 可移植性对内核版本的依赖

eBPF作为一项快速发展的技术，内核增加了很多新的能力，因此，在某个内核上编译好的程序在另一个内核上不一定可以正常运行。

[BPF Features by Linux Kernel Version](https://github.com/iovisor/bcc/blob/master/docs/kernel-versions.md)列出了eBPF在不同内核版本的变化：

* eBPF最早出现在3.15内核中，在3.18才提供bpf()系统调用
* 4.4内核中，任何用户都可以加载eBPF程序
* 4.18引入BTF
* 5.17版本引入CO-RE机制

不同内核版本提供的eBPF的能力不一样，接口也不一样，为了实现程序在不同内核版本之间的迁移性，通常有两种方式：

* [BCC](https://github.com/iovisor/bcc)：直接在高级语言中嵌入内核态代码，当运行程序时直接在本机进行即时编译然后加载到内核，这种方式需要安装编译环境和内核头文件，在某些环境不可行
* [CO-RE](https://nakryiko.com/posts/bpf-core-reference-guide/)：在编译过程中携带字段的偏移量信息，在运行时根据实际的数据进行重定位，从而实现可移植性

### 2 BTF

BTF是一种描述BPF程序使用到的内核数据结构和函数的数据，相当于是一种元数据信息，通过BTF可以去掉BPF程序对内核头文件的依赖。

BTF采用的是二进制格式保存数据，格式为一个btf_header的头部和若干个段[btf.c](https://github.com/torvalds/linux/commit/69b693f0aefa0ed521e8bd02260523b5ae446ad7)：

``` C
struct btf_header {
	__u16	magic; /* 0x9FEB */
	__u8	version; /* 版本号 */
	__u8	flags;

	__u32	parent_label;
	__u32	parent_name;

	/* 下面是各个段在当前文件中的偏移位置(从btf_header末尾开始计算) */
	__u32	label_off;	/* offset of label section	*/
	__u32	object_off;	/* offset of data object section*/
	__u32	func_off;	/* offset of function section	*/
	__u32	type_off;	/* offset of type section	*/
	__u32	str_off;	/* offset of string section	*/
	__u32	str_len;	/* length of string section	*/
};
```

在每个段中就包含一些描述数据，以type section为例，type section用于描述数据结构，当描述一个数据类型时，至少需要知道该数据类型的名称、类型，例如，`struct task_struct`是内核中用于描述进程信息结构体，该数据类型的名称是`task_struct`，该数据类型的类型是`struct`，然后再描述结构体中的字段。

为了描述`struct task_struct`，BTF中用到了两个数据结构：

``` C
/* 用于描述某个数据类型 */
struct btf_type {
    /* 数据类型的名称，例如这里的task_struct，但是这里不直接存储字符串，而是将字符串保存在string section，
     * 这里的name的值就是task_struct这个字符串在string section中的偏移量
     */
	__u32 name;
	/* 描述btf_type表示的类型
	 * bits  0-15: vlen (e.g. # of struct's members)
	 * bits 16-23: unused
	 * bits 24-28: kind (e.g. int, ptr, array...etc)
	 * bits 29-30: unused
	 * bits    31: root
	 */
	__u32 info;
	/* "size" is used by INT, ENUM, STRUCT and UNION.
	 * "size" tells the size of the type it is describing.
	 *
	 * "type" is used by PTR, TYPEDEF, VOLATILE, CONST and RESTRICT.
	 * "type" is a type_id referring to another type.
     * type_id：每个btf_type的对象都会有唯一的值(当然，是对于需要引用其他类型的数据类型)
	 */
	union {
		__u32 size;
		__u32 type;
	};
};

/* struct中的每个成员用btf_member描述， */
struct btf_member {
	__u32	name; /* 与btf_type中的name含义一致 */
	__u32	type; /* 成员的数据类型 */
	__u32	offset;	/* 成员在struct中的以位为单位的偏移量 */
};
```

通过这种方式，BTF文件就可以用于描述数据类型，当然，不同内核版本的BTF实现也不一样，但是，目标都是为了描述BPF程序中使用到的数据结构和函数。

内核在4.18版本引入BTF，后续又提供了`CONFIG_DEBUG_INFO_BTF`选项，当内核开启该选项，通常就会在`/sys/kernel/btf/vmlinux`路径生成BPF程序使用的BTF文件，然后就可以使用`bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h`命令生成`vmlinux.h`，在BPF的内核态程序中引入该头文件，去掉对内核头文件的依赖。

### 3 低版本的内核如何生成BTF

如果内核支持`CONFIG_DEBUG_INFO_BTF`选项，当然是可以直接得到BTF文件，但是，如果内核不支持呢？是否就意味着无法去掉对内核头文件的依赖，必须在机器上安装内核头文件呢？

为了解决低版本的内核无法自动生成BTF文件的问题，[AquaSecurity](https://www.aquasec.com/)对主流的不支持BTF的操作系统生成了BTF文件，生成BTF文件的程序在[BTFHub](https://github.com/aquasecurity/btfhub)，最终生成的BTF文件在[BTFHub Archive](https://github.com/aquasecurity/btfhub-archive)。

[BTFHub Archive](https://github.com/aquasecurity/btfhub-archive)的BTF文件中，总共有8类操作系统，每个操作系统还包含不同的发行版本和架构，某个发行版本和架构还可能会包含多个内核版本，因此，即便单个BTF文件解压后在20MB左右，所有的BTF文件加起来超过50GB，实际分发起来比较困难。一种方式是将这些BTF文件放在集中的存储系统中，当某个机器需要BTF文件时，根据对应的发型版本、架构、内核版本拉取对应的BTF文件，这种方式需要多维护一个存储系统，还会增加额外的网络流量。

比较好的方式是使用`bpftool gen min_core_btf`生成精简的BTF文件，内核有很多数据结构和函数，但是我们的BPF程序通常只使用很少一部分，因此，可以通过BPF的内核态程序结合BTF文件生成精简的必须的BTF文件，文件大小相隔1万倍，同时，虽然内核在变化，但是可能我们的BPF程序使用的这部分很少变化，所以，可能某个发型版本的大版本生成的精简的BTF文件是一模一样的，这无形中也减少了BTF文件的个数，通过这两种方式，整个BTFHub仓库的BTF文件大小可以降到几MB，完全可以打包在客户端程序中进行分发。

`bpftool gen min_core_btf INPUT OUTPUT OBJECT`：

* INPUT：输入的原始的BTF文件
* OUTPUT：输出的精简的BTF文件
* OBJECT：BPF的内核态程序编译完成的目标文件

### 4 CO-RE和BTF

CO-RE表示编译一次、随处运行(Compile Once, Run Everywhere)，表示eBPF对可移植性的整体支持，而BTF表示BPF类型格式(BPF Type Format)，是实现CO-RE中的关键步骤。

现在，高内核版本和低内核版本都有BTF文件，如何使用BTF文件，并实现CO-RE呢？

当使用libbpf库进行开发时，用户态程序的套路通常是：

* bpf_object__open：打开内核态目标文件
* bpf_object__load：加载内核态目标文件
* bpf_program__attach：将内核态目标文件挂载到hook点
* perf_buffer__new：创建perf_buffer，并指定数据的回调函数
* perf_buffer__poll：接收内核发送的数据
* perf_buffer__free：释放perf_buffer
* bpf_object__close：关闭内核态目标文件

这里的第一步打开内核态目标文件还可以用其他方式：

* bpf_object__open_file：与bpf_object__open类似，区别是多了个opts参数
* bpf_object__open_mem：目标文件已经读取到内存，直接从内存获取目标文件，也有opts参数

上面两个函数都有`const struct bpf_object_open_opts *opts`参数，在该结构体中可以指定BTF文件的路径：`const char *btf_custom_path`，因此，如果机器本身并不支持BTF，就可以直接用生成的精简的BTF文件，将文件路径放到opts中的btf_custom_path中。在执行`bpf_object__load`加载内核态目标文件时，会使用目标机器的BTF文件进行重定位，例如，在使用clang编译内核态代码时，根据编译时的vmlinux.h记录类型的偏移信息，在实际加载内核态目标文件时，根据目标机器的BTF文件的字段进行重定位，转换为正确的字段偏移，从而实现CO-RE。

为了实现CO-RE，依赖以下能力：

* BTF提供对内核的数据类型的描述能力，用一种独立的文件格式记录内核的数据类型
* clang编译器提供记录重定位信息的能力
* libbpf在加载内核态目标文件时根据内核的BTF文件进行重定位，达到适配目标机器的能力

不同内核造成的影响主要是字段的区别，例如，字段偏移有区别，字段名改了，因此，在BPF程序中，CO-RE主要针对的就是内核数据读取的接口，libbpf则针对有BTF和没有BTF提供了两套接口。

### 5 bpf_core_read

在BPF的内核态程序中，当需要获取内核的数据时，需要引用libbpf的`bpf_core_read.h`头文件，该头文件中的函数就是用于读取内核的字段。

当需要获取pid时，在bcc中可以直接使用`pid_t pid = task->pid;`，但是，使用libbpf就无法这么使用。

在该头文件中，有个函数与头文件的文件名相同：`bpf_core_read`，使用该函数也可以实现读取pid：

``` C
struct task_struct *task = (struct task_struct*)bpf_get_current_task();
pid_t pid;
bpf_core_read(&pid, sizeof(pid), &task->pid);
```

查看`bpf_core_read`的定义，其实是个宏：

``` C
#define bpf_core_read(dst, sz, src)					    \
	bpf_probe_read_kernel(dst, sz, (const void *)__builtin_preserve_access_index(src))
```

看`bpf_core_read`的三个参数，可以理解为：读取src的sz个字节，将数据拷贝到dst。`bpf_core_read`的实现使用了`bpf_probe_read_kernel`，对src调用`__builtin_preserve_access_index`然后传递给`bpf_probe_read_kernel`。

`__builtin_preserve_access_index`是clang编译器提供的用于偏移量重定位的函数，通过该函数封装的地址在实际执行时就可以根据BTF提供的信息进行字段偏移量的调整。

所以，对于兼容性要求比较高的程序来说，需要提供有BTF和没有BTF的两种实现方式：当有BTF时，可以调用`bpf_core_read`函数；当没有BTF时，需要调用`bpf_probe_read_kernel`。

除了`bpf_core_read`和`bpf_probe_read_kernel`，`bpf_core_read.h`中还提供了很多以`BPF_CORE_READ`和`bpf_core_read`开头的宏，都是为了方便读取内核的数据或者是从用户态传递给内核态的数据。

### 6 参考文档

* [ebpf中的BTF与CO-RE](https://segmentfault.com/a/1190000044379552)
* [BPF编程-使用libbpf-bootstrap构建BPF应用程序【译】](https://blog.csdn.net/sinat_38816924/article/details/122259826)
* [BTF Hub](https://github.com/aquasecurity/btfhub)
* [BPF 可移植性和 CO-RE](https://arthurchiao.art/blog/bpf-portability-and-co-re-zh/)
* [BTFGen: 让 eBPF 程序可移植发布更近一步](https://developer.aliyun.com/article/899354)
