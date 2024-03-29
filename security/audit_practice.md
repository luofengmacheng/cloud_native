## audit的一些问题以及需要注意的地方

### 1 audit存在的一些问题

#### 1.1 audit_cmd_mutex锁占用的问题

当内核生成审计日志后，会以单播形式发送给用户态的某个进程，因此，某个程序如果想要接受审计日志，需要先调用audit_set_pid，然后使用select+recv接收审计日志。

当内核在执行audit_set_pid时，内核会占用audit_cmd_mutex，然后判断是否已经有其他进程占用audit，如果有就会尝试发送消息，从而判断用户态进程在正常接收审计日志，那么该调用就会返回File exists的报错。而在发送消息之前会判断接收缓冲区是否满，如果满，则会进入睡眠状态且不会被唤醒，由于占用了audit_cmd_mutex锁，也会造成bash、sshd等进程进入不可中断状态，系统会如卡死一样。

该问题出现在3.10.0-514到3.10.0-1160，对应了CentOS的7.3~7.9，在CentOS 8修复。

因此，在存在该问题的系统上，不能先执行audit_set_pid，然后通过结果判断，而是应该先干掉占用的进程，只在audit_pid为0时才调用audit_set_pid。

#### 1.2 60秒睡眠的问题

audit有个backlog_limit参数，含义是内核的audit缓存队列上限。当用户态程序消费审计日志过慢，造成缓存队列满时，会触发内核的睡眠机制，造成系统卡顿。

睡眠时间由backlog_wait_time参数控制，该值默认是60秒。当内核版本小于3.14时，该参数不能设置，当内核版本大于3.14时，可以通过接口设置为0，让内核不睡眠。

#### 1.3 systemd-journald-audit.socket服务导致磁盘满的问题

部分系统上会默认开启systemd-journald-audit.socket服务，该服务会接收审计日志并将审计日志写入系统日志，如果系统日志是按周轮转，可能在还未轮转时就将磁盘打满。一种解决办法是停掉该服务。

### 2 需要注意的地方

* failure：该参数会影响audit遇到错误时的处理行为，0表示什么也不干，1表示输出日志到系统日志，2表示系统崩溃，默认值为1，为了保证系统的稳定，建议设置为0
* backlog_limit：audit的缓存队列大小，如果设置太小会导致频繁丢弃日志
* backlog_wait_time：缓存满的休眠时间，建议设置为0
* 审计日志解析：为了提高系统的日志处理量，在解析日志时建议引入缓存队列和丢弃机制，并且需要快速执行结束，否则会造成日志丢弃
* 路径监控的读写分离：在进行路径监控时可以设置需要的操作类型，有4种，分别是读、写、执行、权限变更，对于读而言，如果设置某些路径（例如，/etc/），会导致日志量非常大，解决方案是如果不需要读，则将读去掉，如果需要读，尽量精确到子路径
