## 通过/proc/pid/获取进程信息

### 1 关于/proc

/proc是一个内存文件系统，它保存了系统运行的信息，比如，系统启动时间、内存使用率等，同时，对于每个进程，都会有一个目录保存该进程的信息。

### 2 进程的基本信息

这里不会列出所有的文件，只列出部分文件：

* cmdline：命令行，注意：命令行中的空格会以空字符表示，因此，如果是程序读取命令行时，需要读取整个文件，然后遍历所有字符，将空字符转换为空格
* comm：进程名
* cwd：当前工作目录，是个软链接，指向实际的路径
* environ：环境变量
* exe：进程启动的二进制，也是个软链接，指向实际的文件路径
* fd：进程打开的文件描述符，每个描述符也是个软链接，指向打开的文件，如果涉及到socket，则会显示socket的inode号
* fdinfo/：进程打开文件时的一些属性
* limits：进程的资源限制，内容与`ulimit -a`类似
* maps：内存占用的地址空间以及对应的文件
* root：根路径
* stack：进程的堆栈
* stat：进程的状态信息，包括ppid、进程名、进程启动时间等
* status：进程的一些参数信息，包括uid、gid、虚拟内存、capability等
* task/：进程的子线程，每个子目录就是一个线程的信息

### 3 进程的容器相关信息

进程的信息中跟容器相关的主要有两个：

* 容器ID：container_id
* 进程在容器中的pid：nspid

如果是容器中的进程，在/proc/pid/cgroup中可以看到容器ID：

``` shell
-bash-4.2# cat /proc/10135/cgroup
11:hugetlb:/docker/9078ff447fb7ade9339bea248a38b8dad1f1745f197d1a10a3c7097a088be5e8
10:cpuacct,cpu:/docker/9078ff447fb7ade9339bea248a38b8dad1f1745f197d1a10a3c7097a088be5e8
9:perf_event:/docker/9078ff447fb7ade9339bea248a38b8dad1f1745f197d1a10a3c7097a088be5e8
8:freezer:/docker/9078ff447fb7ade9339bea248a38b8dad1f1745f197d1a10a3c7097a088be5e8
7:memory:/docker/9078ff447fb7ade9339bea248a38b8dad1f1745f197d1a10a3c7097a088be5e8
6:devices:/docker/9078ff447fb7ade9339bea248a38b8dad1f1745f197d1a10a3c7097a088be5e8
5:pids:/docker/9078ff447fb7ade9339bea248a38b8dad1f1745f197d1a10a3c7097a088be5e8
4:cpuset:/docker/9078ff447fb7ade9339bea248a38b8dad1f1745f197d1a10a3c7097a088be5e8
3:net_prio,net_cls:/docker/9078ff447fb7ade9339bea248a38b8dad1f1745f197d1a10a3c7097a088be5e8
2:blkio:/docker/9078ff447fb7ade9339bea248a38b8dad1f1745f197d1a10a3c7097a088be5e8
1:name=systemd:/docker/9078ff447fb7ade9339bea248a38b8dad1f1745f197d1a10a3c7097a088be5e8
```

这里面是每个cgroup的路径，每行的最后的64个字符组成的串就是容器ID，执行`docker ps --no-trunc`可以看到该容器ID对应的容器。

nspid的获取跟内核版本相关：

* 如果内核版本小于4.1，则只能通过容器中的进程的nspid的sched得到宿主机的pid，也就是说，需要进入到容器，然后
* 如果内核版本大于或者等于4.1，可以读取/proc/pid/status，其中有一行以Nspid开头，一般只有两个值，第一个值是该进程在宿主机上的pid，第二个值是该进程在容器中的pid，也就是nspid

``` shell
-bash-4.2# docker exec -it 9078ff447fb7ade9339bea248a38b8dad1f1745f197d1a10a3c7097a088be5e8 sh
sh-3.2# cat /proc/1/sched
bash (10135, #threads: 1)
-------------------------------------------------------------------
```

上面就是进入到容器中，然后查看nspid为1的进程的sched文件就可以查询到该进程在宿主机上的pid，但是无法在宿主机上查询到容器中的nspid。

### 4 进程的网络相关信息

/proc/pid/fd目录是程序中使用的文件描述符，如果打开的是文件，则指向文件，如果打开的是设备，则指向设备文件，如果打开的是网络，也就是socket，指向的则是一个inode：

``` shell
-bash-4.2# ls -l /proc/1/fd
total 0
lrwx------. 1 root root 64 Nov 16 17:34 0 -> /dev/null
lrwx------. 1 root root 64 Nov 16 17:34 1 -> /dev/null
lrwx------. 1 root root 64 Nov 16 17:37 100 -> socket:[39232]
lrwx------. 1 root root 64 Nov 16 17:37 101 -> socket:[36748]
```

如上所示，100和101分别指向socket，如何根据socket的inode获取连接信息呢？

/proc/pid/net中是进程的网络相关数据，里面会按照协议分类展示出数据，需要注意的是，有些协议是全局的，但是也会在这里有对应的数据，例如，arp是全局的，/proc/pid/net/arp是当前进程使用的arp信息，可以发现所有进程的该文件内容都是一样的。

/proc/pid/net常见的内容有：

* arp：arp -a
* netlink：netlink的连接
* route：route -n
* tcp/tcp6：TCP的ipv4和ipv6连接
* udp/udp6：UDP的ipv4和ipv6连接
* unix：unix套接字的连接

/proc/pid/net/tcp的文件格式类似以下结构：

``` shell
  sl  local_address rem_address   st tx_queue rx_queue tr tm->when retrnsmt   uid  timeout inode                                                     
   0: 00000000:006F 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 22741 1 ffff9ef170b80000 100 0 0 10 0                     
   1: 0100007F:8892 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 84466 1 ffff9ef170b826c0 100 0 0 10 0                     
   2: 017AA8C0:0035 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 37480 1 ffff9ef164f38000 100 0 0 10 0                     
```

这里面比较重要的就是local_address和rem_address，分别以十六进制显示本地和远端的IP地址和端口，后面还有个字段是inode，就对应`/proc/1/fd`中的inode，另外的tcp、udp、netlink也可以通过一样的方式找到对应的连接的具体信息。

### 5 总结

/proc/pid中包含了进程的许多信息，这些信息可以供用户查看，也可以供程序读取，程序可以从这里获取到进程的文件、网络等信息并进行分析，但是，频繁读取也可能影响业务性能。
