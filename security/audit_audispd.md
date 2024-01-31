## audispd调研

### 1 问题背景

在Linux中，当某个进程调用audit_set_pid将自己的pid保存到内核的audit模块后，如果有日志生成，kaudit内核线程就会通过netlink通信机制将审计日志发送给audit_pid，因此，只能有一个进程占用audit并接收audit日志，那如果有另一个进程已经占用audit呢？例如，auditd或者其他组件。

一种方式是可以将选择权交给用户，用户可以看到当前占用audit的进程，然后再决定是否将已经占用audit的进程杀死，然后进行占用。另一种方式是，如果用户已经在使用auditd接收日志，是否有一种方式可以让auditd在运行的同时我们也能够接收和处理日志呢？这种场景下就可以使用audispd。

### 2 audispd

audispd是作为auditd的子进程运行，而audispd会开启一些插件用于接收审计日志，例如，syslog或者sedispatch等，也就是说，日志转发路径变成了：

kauditd -> auditd -> audispd -> af_unix/syslog/sedispatch

下面逐个看下配置文件。

auditd.conf：

``` conf
local_events = yes
write_logs = yes // 将日志写入磁盘
log_file = /var/log/audit/audit.log
log_group = root
log_format = RAW
flush = INCREMENTAL_ASYNC // 刷新模式，当前为异步增量刷新模式
freq = 50 // 接收到多少条日志后就写入磁盘
max_log_file = 8 // 日志文件大小，8MB
num_logs = 5 // 最大文件个数
priority_boost = 4 //
disp_qos = lossy
dispatcher = /sbin/audispd // 将审计日志转发的目的进程的文件名
```

以上是auditd.conf中的部分配置项，除了控制auditd写入日志文件的配置项，还有与日志转发相关的两个配置项：dispatcher，disp_qos，其中，dispatcher是负责日志转发的组件，auditd在启动后会拉起该程序，然后将收到的日志转发给该程序，该程序可以从标准输入读取审计日志，disp_qos则是控制当审计日志太多时auditd和dispatcher之间的行为，auditd和dispatcher之间有一个128K的队列，当队列满时，日志会被丢弃(lossy)或者auditd会等待队列有空间可以放入(lossless)。

audispd.conf：

``` conf
q_depth = 250 // 队列深度
overflow_action = SYSLOG // 当内部队列满时的行为，当前值表示会写入一条syslog的日志
priority_boost = 4 //
max_restarts = 10 // 重启插件的最大次数
name_format = HOSTNAME // 节点信息如何插入日志中，当前值表示会将HOSTNAME写入到日志中
plugin_dir = /etc/audisp/plugins.d/ // 插件的路径
```

在plugin_dir目录下存放的就是插件的配置，每个配置项告诉audispd应该如何转发日志，例如，af_unix.conf：

``` conf
active = no // 是否启用，yes/no
direction = out //
path = builtin_af_unix // 插件二进制的绝对路径，对于内部插件，就是插件的名称，内部插件有af_unix和syslog
type = builtin // 类型，builtin/always
args = 0640 /var/run/audispd_events // 参数
format = string // 格式，binary/string
```

从插件类型看，总共有三种类型：

内部插件的af_unix：audispd将审计日志发送给unix套接字
内部插件的syslog：audispd将审计日志发送给syslog
外部插件：audispd将审计日志发送给某个程序的标准输入

从上面对几个配置文件进行分析可以发现，auditd和audispd只是作为审计日志转发的中间组件，并且使用队列作为中间缓冲，而对于内核的日志发送者来说，日志的接收者只有auditd。

从几个组件的启动方来看，auditd负责拉起audispd，audispd负责拉起always类型的插件。

因此，如果在某些场景下希望与auditd同时运行，且获取审计日志，可以使用audispd的af_unix插件，但是可能存在其他问题：

* 规则是全局的，多个程序都只设置自己的规则，每个程序都会收到所有的数据
* 对于新增af_unix插件的操作，如果插件的目录是默认的，当然没有问题，如果是其他路径，可能无法获取到插件目录
* audispd增加新的af_unix插件后，需要重启auditd才能生效，如果agent启动一段时间后出现问题退出就会导致频繁kill auditd，会造成启动失败
* 如果其他非auditd程序正在使用audit，那就无法使用audispd
* 如果audispd没有启动，可能是由于启动异常，也可能是由于没有需要启动的插件，如何区分

### 3 高版本中的audispd

如何查看auditd的版本号？auditd自身没有提供查看版本号的选项，只能通过日志或者包管理工具查看auditd的版本号。

CentOS中，在/var/log/messages中会打印auditd的版本号，也可以通过`rpm -qi audit`查看版本号。对于ubuntu类，在/var/log/kern.log中没看到auditd的版本号，但是可以通过`dpkg -l | grep auditd`查看版本号。

audit-userspace仓库的标签最早只有2.7.4，这个版本已经有audispd，而从3.0.0开始audispd作为auditd的一个线程工作，因此，auditd就包含了audispd原有的能力。

### 4 audispd引入的性能问题

由于使用audispd进行日志中转，在中转过程中引入了两个队列：

* auditd和audispd之间有一个128K的缓存队列，disp_qos用于控制缓存队列满的行为
* audispd内部会维护一个队列，q_depth用于控制队列的长度

测试环境：

* CentOS 7.5(3.10.0-1160.102.1.el7.x86_64)
* 4C8G
* audit: backlog_limit=8192

只启动auditd，audispd没有配置插件：

是否丢弃日志可以通过查看`auditctl -s`中的lost或者/var/log/messages日志文件。

| 事件量级(eps) | 是否丢弃日志 | 系统CPU |
| ------- | ----------- | ------- |
| 1000 | NO | 1.3% |
| 5000 | NO | 5% |
| 6000 | NO | 6% |
| 7000 | NO | 7% |
| 8000 | YES | 8% |
| 9000 | YES | 10% |
| 10000 | YES | 10% |

启动auditd，audispd开启af_unix插件并使用socat命令接收审计日志：

使用`socat - UNIX-CONNECT:/var/run/audispd_events`命令接收审计日志，是否丢弃日志可以查看/var/log/messages中的`dispatch err (pipe full) event lost`日志。

| 事件量级(eps) | q_depth |是否丢弃日志 | 系统CPU | 备注 |
| ------- | ------- | ----------- | ------ | ---- |
| 500 | 80  | NO | 2% | |
| 500 | 250  | NO | 2% | |
| 500 | 5000  | NO | 2% | |
| 1000 | 80  | YES | 4% | |
| 1000 | 250  | YES | 4% | |
| 1000 | 5000  | YES | 4% | 用例执行3分钟开始丢弃日志 |

不启动auditd，使用其他程序占用并接收审计日志：

``` C
#include <stdio.h>
#include <libaudit.h>
#include <errno.h>

int main() {
    pid_t pid = getpid();
    printf("current pid=%d\n", pid);

    int fd = audit_open();
    int ret = 0;
    ret = audit_set_pid(fd, pid, WAIT_YES);
    if(ret<0) {
        perror("audit set_pid failed: ");
    }
    ret = audit_set_backlog_limit(fd, 8192);
    if(ret<0) {
        perror("audit_set_backlog_limit failed: ");
    }

    struct audit_reply audit_rep;

    struct sockaddr_nl local_addr;
    memset(&local_addr, 0, sizeof(local_addr));
    local_addr.nl_family = AF_NETLINK;

    do {
        fd_set read_mask;
        FD_ZERO(&read_mask);
        FD_SET(fd, &read_mask);
        ret = select(fd + 1, &read_mask, NULL, NULL, NULL);
        if(ret < 0 ) {
            break;
        }
        ret = audit_get_reply(fd, &audit_rep,
                              GET_REPLY_NONBLOCKING, 0);
        audit_rep.msg.data[audit_rep.len] = '\0';
        printf("recv audit msg: %s\n", audit_rep.msg.data);
    } while(1);

    audit_close(fd);
}
```

如果保留上面的代码中的printf语句，事件量级只能到1000eps，2000eps就开始丢弃日志；如果将printf语句注释，事件量级则可以到9000eps。

从上面的测试结果可以得出以下结论：

* auditd和audispd之间的队列对目标程序接收审计日志的影响很大，完全限制了处理的审计日志量级
* audispd内部的q_depth参数对审计日志量级影响不大，测试时可以忽略
* 性能排序：直接读取审计日志并快速处理 > 启动auditd但不启动audispd > 启动audispd转发日志

### 5 audispd源码解析

auditd中与audispd相关的逻辑有两个地方：

* audispd的拉起
* auditd接收审计日志并将审计日志发送给audispd

在auditd的main函数中，会调用init_dispatcher()拉起dispatcher，也就是audispd。

init_dispatcher()会干下面三件事：

* 使用socketpair创建两个socket保存到disp_pipe
* 根据qos选项设置socket的O_NONBLOCK：如果qos设置为lossy，则将disp_pipe中的socket设置为O_NONBLOCK，否则就不设置
* 使用fork和execl在子进程启动dispatcher，并且在执行execl之前将disp_pipe[0]设置到stdin上，这样dispatcher的stdin就是disp_pipe[0]

拉起audispd后，会执行audit_set_pid占用audit，然后设置audit fd的事件处理函数netlink_handler。在netlink_handler中会调用audit_get_reply读取数据，在收到审计日志后调用distribute_event进行事件的分发。distribute_event调用dispatch_event发送审计日志，并且调用handle_event将日志写入到本地磁盘。

dispatch_event()调用writev将数据写入到dispatcher，由于disp_pipe[0]对应的是dispatcher的stdin，因此，调用writev将数据写入disp_pipe[1]时，dispatcher就可以收到数据，在多次尝试后失败就会进行错误处理，此处有三种情况：

* 如果错误码是EAGAIN并且是第一次调用dispatch_event，则直接返回，因此，可以在第二次再次调用
* 如果失败数量少于REPORT_LIMIT，则打印dispatch err () event lost的错误信息
* 如果失败数量到达REPORT_LIMIT，则打印`dispatch error reporting limit reached - ending report notification.`

打印错误信息后，如果失败数量继续增加则不会继续打印，除非某次发送数据成功，错误计数器清0，重新开始计算。

在audispd中，会调用`audit_fd=dup(0)`得到用于接收数据的文件描述符，再读取plugins.d目录下的插件，然后启动各个插件，接下来就是注册事件回调和事件处理：

* add_event()注册audit_fd的POLLIN事件，以及事件的回调函数process_inbound_event
* 创建子线程inbound_thread_main，在子线程中调用poll()判断是否有fd可读，然后调用对应的fd的回调函数，对于audit_fd来说，就会调用process_inbound_event
* process_inbound_event()调用readv读取数据，然后将数据放到队列中
* main()中的event_loop()从队列中读取数据，然后遍历所有的插件，根据插件类型调用不同的发送函数：如果是syslog，则调用send_syslog发送；如果是af_unix，则调用send_af_unix发送；如果是always，则调用wirte_to_plugin发送

从代码中只看到audispd中队列的处理，先将事件放到队列中，然后再从队列中读，对于auditd和audispd之间的缓冲应该是依赖socketpair创建的socket的缓冲区，但是，socket的缓冲区大小是由系统配置决定的，而且auditd的代码中也没看到调用setsockopt设置缓冲区大小的代码。

### 6 结论

audispd能够将auditd收到的日志转发出来供多个程序使用，也让其他程序能够得以与auditd同时运行，但是，使用该方式需要解决auditd的启停问题以及audispd带来的性能损耗问题，因此，如果直接使用audit接收日志不会引起其他的问题，建议还是直接使用占用audit的方式。
