## 云原生运行时安全Falco

### 1 安装和使用

``` shell
curl -L -O https://download.falco.org/packages/bin/x86_64/falco-0.32.2-x86_64.tar.gz

tar -xvf falco-0.32.2-x86_64.tar.gz
cp -R falco-0.32.2-x86_64/* /

yum -y install kernel-devel-$(uname -r)

falco-driver-loader #加载内核驱动
falco-driver-loader bpf #加载ebpf驱动

# 如果要使用ebpf驱动，则要定义以下环境变量
# export FALCO_BPF_PROBE=""
falco
```

直接执行falco命令后就可以在终端看到结果。

### 2 架构

falco是基于规则的运行时安全检测工具，最关键的是拿到用户的行为数据，然后对规则进行处理，如果满足规则条件，则根据规则输出告警数据。

因此，falco可以分为以下几个部分：

* 行为数据采集：通过驱动采集系统调用数据，这部分对应上面的falco-driver-loader和驱动程序，调用falco-driver-loader可以加载驱动程序，驱动程序有两种：内核模块和eBPF
* 规则文件：执行falco时，falco会加载配置文件，配置文件默认路径是/etc/falco/falco.yaml，在配置文件中会定义要加载的rules_file
* 规则判断并输出告警：执行falco时，在加载完规则文件后，会开启一个webserver，并监听端口，负责接收驱动程序发送来的数据

这里面falco的逻辑相对比较好理解，就是解析规则，根据收到的数据匹配规则，如果满足规则，说明命中一个内部定义的异常，则发送告警，因此，难点就是驱动程序。

falco的代码主要在两个github仓库中，一个是falco，另一个是libs。libs中包含几个模块：

* driver：驱动部分，负责内核系统调用的实际数据采集，其中最外层是kernel module，2个子目录分别是bpf和modern_bpf
* userspace/libscap：负责对接数据源，例如从内核模块、bpf、gvisor获取数据
* userspace/libsinsp：

### 3 内核模块 vs eBPF

falco支持内核模块和eBPF获取系统调用数据，它们分别存在各自的优点和缺点，其中，内核模块的优点就是高性能，缺点则是门槛较高，容易把内核搞崩溃；eBPF的有点就是门槛较低，性能比用户态的程序高，而且安全性好，缺点则是程序的编译和分发。当然，两者都有内核版本的适配问题，内核模块需要对每个版本的内核进行编译，而ebpf由于发展较快，api变化也较快，同样有内核版本的适配问题。

因此，使用eBPF需要解决两个问题：
* 内核版本的适配问题
* eBPF的编译和分发

### 4 eBPF的编译和分发

当前eBPF的编译和分发的方式：

* BCC：
* BTF：
* BumbleBee：

有用的一些ebpf仓库：
(1) https://github.com/ehids/ebpf-slide
(2) 