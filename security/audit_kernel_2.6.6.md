## 2.6.6版本的audit审计机制分析

### 1 关于内核版本

linux内核从2.6.6版本开始支持audit机制，为了更好的理解audit本身的机制，需要对audit的内核代码进行分析。

### 2 入口

内核中，audit相关的代码主要有三个文件：

* include/linux/audit.h 头文件，主要包含常量、数据结构和接口声明
* kernel/audit.c 全局变量定义以及接口的实现
* kernel/auditsc.c 系统调用的审计的实现

从audit的功能来说，包含三个部分：

* audit的初始化
* 审计日志的生成和写入
* 规则以及其他的操作函数

### 3 audit初始化

audit的初始化包括两个函数：

* audit_init()，该函数用__initcall宏进行修饰，该函数会被放到.init.text段，在audit模块加载时会被调用，是整个audit模块的初始化函数
* audit_enable()，该函数用__setup宏进行修饰，当bootloader传给kernel的参数是audit=0或者audit=1，就会调用audit_enable()函数，并将0或者1作为字符串参数传给audit_enable()

对于audit_init()函数，根据CONFIG_NET宏还定义了不同的实现，如果定义了CONFIG_NET，就会创建aduit的netlink的socket，否则，就不会创建socket。    

### 4 系统调用审计流程

当前的audit版本只支持系统调用的审计，因此，主要查看系统调用的审计流程。

系统调用的入口位于arch/x86_64/kernel/entry.S，中间的x86_64随实际的架构有所变化。

``` x86asm
ENTRY(system_call)
	CFI_STARTPROC
	swapgs
	movq	%rsp,%gs:pda_oldrsp 
	movq	%gs:pda_kernelstack,%rsp
	sti					
	SAVE_ARGS 8,1
	movq  %rax,ORIG_RAX-ARGOFFSET(%rsp) 
	movq  %rcx,RIP-ARGOFFSET(%rsp)  
	GET_THREAD_INFO(%rcx)
	testl $(_TIF_SYSCALL_TRACE|_TIF_SYSCALL_AUDIT),threadinfo_flags(%rcx)
	jnz tracesys
	cmpq $__NR_syscall_max,%rax
	ja badsys
	movq %r10,%rcx
	call *sys_call_table(,%rax,8)  # XXX:	 rip relative
	movq %rax,RAX-ARGOFFSET(%rsp)
```

上面是系统调用的入口，开始会处理一些参数，中间就会判断是否启用trace或者audit，如果启用，就会调用tracesys：

``` x86asm
tracesys:			 
	SAVE_REST
	movq $-ENOSYS,RAX(%rsp)
	FIXUP_TOP_OF_STACK %rdi
	movq %rsp,%rdi
	call syscall_trace_enter
	LOAD_ARGS ARGOFFSET  /* reload args from stack in case ptrace changed it */
	RESTORE_REST
	cmpq $__NR_syscall_max,%rax
	ja  1f
	movq %r10,%rcx	/* fixup for C */
	call *sys_call_table(,%rax,8)
	movq %rax,RAX-ARGOFFSET(%rsp)
1:	SAVE_REST
	movq %rsp,%rdi
	call syscall_trace_leave
	RESTORE_TOP_OF_STACK %rbx
	RESTORE_REST
	jmp ret_from_sys_call
```

这里就进行系统调用的审计，里面有三个call，中间的call就是调用系统调用，第一个call是系统调用入口的审计，第三个call是系统调用出口的审计。

``` C
// arch/x86_64/kernel/ptrace.c
asmlinkage void syscall_trace_enter(struct pt_regs *regs)
{
	if (unlikely(current->audit_context))
		audit_syscall_entry(current, regs->orig_rax,
				    regs->rdi, regs->rsi,
				    regs->rdx, regs->r10);

	if (test_thread_flag(TIF_SYSCALL_TRACE)
	    && (current->ptrace & PT_PTRACED))
		syscall_trace(regs);
}
```

这里的unlikely可以进行编译优化：[Linux内核入门-- likely和unlikely](https://blog.csdn.net/m0_74282605/article/details/127749787)。

后续调用流程如下：

* audit_syscall_entry 位于kernel/auditsc.c
    * audit_alloc_context 创建audit_context
    * audit_filter_syscall
        * audit_filter_rules 对比进程的信息和规则确定是否匹配，此处实现的就是`-F`的过滤选项，过滤选项可以对产生日志的进程的字段进行过滤，该函数的返回值表明过滤选项是否匹配上，同时，会根据action设置state
    * 如果规则判断不是AUDIT_DISABLE，则将serial、time保存到context中，并在context中保存两个标记：in_syscall和auditable

系统调用结束时的调用流程：

* audit_syscall_exit 位于kernel/auditsc.c
    * audit_get_context
        * 从task_struct中获取audit的context
        * audit_filter_syscall
        * `用进程的信息填充context`，例如，填充pid：`context->pid = tsk->pid`
    * audit_log_exit 如果context中的in_syscall==1和auditable==1，则执行
        * audit_log_start
            * 如果audit_backlog大于audit_backlog_limit并且audit_rate_check()为真，则内核日志会出现`audit: audit_backlog=XX > audit_backlog_limit=XX`的打印
            * audit_log_lost audit_lost加1：`atomic_inc(&audit_lost)`，如果rate_limit为0，则会在内核日志中打印`audit: audit_lost=XX audit_backlog=XX audit_rate_limit=XX audit_backlog_limit=XX`
            * 从audit_freelist链表中取一个audit_buffer，如果获取不到，则分配一个，如果分配失败，则调用audit_log_lost
            * audit_backlog加1：`atomic_inc(&audit_backlog)`
            * audit_log_format 将`audit(second.milisecond:serial)`写入刚才分配的audit_buffer
        * audit_log_format 先将syscall、per、exit、参数以及一些id作为审计日志输出，然后再输出`系统调用过程中访问的文件和设备信息`
            * audit_log_vformat 将要审计日志写入audit_buffer
        * audit_log_end
            * audit_log_end_irq 如果当前`处于硬件中断上下文`时调用，将audit_buffer放到audit_txlist队列
                * list_add_tail
                * tasklet_schedule(&audit_tasklet) 进行tasklet的任务调度，执行audit_tasklet
            * audit_log_end_fast 如果当前`不处于硬件中断上下文`时调用，直接将audit_buffer中的数据发送到用户空间
                * audit_log_move 将audit_buffer中的内容拷贝到sk_buff
                * audit_log_drain 遍历audit_buffer中的sk_buff，将sk_buffer发送到用户空间
                    * 从audit_buffer中的sklist队列获取sk_buff，然后根据audit_pid将调用`netlink_unicast`将数据发送出去，如果多次发送失败，且错误码是EAGAIN，就会调用audit_log_end_irq将audit_buffer放到audit_txlist队列，其他错误则会打印错误日志；如果发送数据的返回值是ECONNREFUSED，则在内核日中打印`audit: *NO* daemon at audit_pid=%d`，并将audit_pid设置为0
                    * 如果audit_pid为0，也就是没有进程订阅audit，则会将审计日志输出到内核日志
                * audit_backlog减1：`atomic_dec(&audit_backlog)`
                * kfree/list_add：如果当前空闲链表的节点过多，则直接将audit_buffer释放，否则将audit_buffer放到空闲链表
    * 将context中的in_syscall和auditable都设置为0
    * 释放context

从整个实现的流程看，主要依赖audit_context进行数据传递，在真正要执行系统调用前，创建audit_context，将参数保存到audit_context，并生成audit日志的`audit(second.milisecond:serial)`部分（保证单次系统调用生成的多条审计日志的这部分内容是一样的），将audit_context保存到进程的task_struct，在系统调用结束后，获取进程的audit_context，用进程的信息填充audit_context，再将数据写入到audit_buffer缓存中，最后要么直接通过netlink_unicast发送给用户态进程，要么将数据放到audit_txlist队列。

audit_log_exit()会将audit_context中的数据输出到审计日志：

``` C
struct audit_names {
	const char	*name;
	unsigned long	ino;
	dev_t		rdev;
};

struct audit_context {
	int		    in_syscall;	/* 标记是否正在执行系统调用 */
	enum audit_state    state;
	unsigned int	    serial;     /* 审计日志的序列号 */
	struct timespec	    ctime;      /* 系统调用入口的时间 */
	uid_t		    loginuid;   /* login uid (identity) */
	int		    major;      /* 系统调用号 */
	unsigned long	    argv[4];    /* 系统调用参数 */
	int		    return_valid; /* 返回值是否有效 */
	int		    return_code;/* 系统调用返回值 */
	int		    auditable;  /* 标记是否需要写入审计日志 */
	int		    name_count;
	struct audit_names  names[AUDIT_NAMES];
	struct audit_context *previous; /* 系统调用嵌套 */

	/* 写入审计日志的字段 */
	pid_t		    pid;
	uid_t		    uid, euid, suid, fsuid;
	gid_t		    gid, egid, sgid, fsgid;
	unsigned long	    personality;

#if AUDIT_DEBUG
	int		    put_count;
	int		    ino_count;
#endif
};
```

在打印日志时，会先输出一条日志，里面会输出pid到personality这些字段，以及上面的liginuid、major、argv、return_code等字段，然后就会打印names这个数组中的字段，这个里面是什么呢？看字面意思就是名字，是什么的名字呢？而且里面还有name、inode、rdev这些字段。其实，这里保存的就是系统调用执行过程中访问的文件或者目录的信息。

有两个地方会向names中加入元素：

* fs/namei.c中的getname()：将用户空间的文件名拷贝到内核空间，然后将文件名保存到audit_names
* fs/namei.c中的path_lookup()调用audit_inode()将查找的路径的dentry的inode的i_ino和i_rdev保存到audit_names

names中的元素打印的格式类似于`item=0 name=fname inode=11111 dev=xx:xx`，因此，这里打印的就是系统调用过程中查找或者访问过的目录或者文件。

### 5 配置下发流程

上面是系统调用过程中，根据给定的规则然后输出审计日志，那么，审计规则是怎么来的呢？当用户在机器上执行`auditctl -l`或者`auditctl -S`进行审计规则的读取和设置时，依然是通过NETLINK_AUDIT的netlink socket进行通信的。audit自身的一些配置也是采用同样的流程。

当用户态程序需要增加规则时，通常会使用libaudit的audit_add_rule_data()：

``` C
int audit_add_rule_data(int fd, struct audit_rule_data *rule,
        int flags, int action) {
    int rc;

    if (flags == AUDIT_FILTER_ENTRY) {
        audit_msg(LOG_WARNING, "Use of entry filter is deprecated");
        return -2;
    }
    rule->flags = flags;
    rule->action = action;
    rc = audit_send(fd, AUDIT_ADD_RULE, rule,
            sizeof (struct audit_rule_data) +rule->buflen);
    if (rc < 0)
        audit_msg(audit_priority(errno),
            "Error sending add rule data request (%s)",
            errno == EEXIST ?
            "Rule exists" : strerror(-rc));
    return rc;
}
```

这里的audit_send就是调用sendto系统调用向内核发送规则数据，调用sendto时，addr.nl_family设置为AF_NETLINK，下一步就会进入到内核的sys_sendto系统调用。

sys_sendto的调用流程如下：

* sys_sendto
    * sock_sendmsg
        * __sock_sendmsg
            * sock->ops->sendmsg 调用套接字的sendmsg函数，此处不同的family类型就会调用不同的函数，对于AF_NETLINK来说就会调用netlink_sendmsg
                * netlink_unicast
                    * netlink_attachskb
                    * netlink_sendskb
                        * sk->sk_data_ready 此处将sk_buff放到socket的接收队列的尾部，然后调用sk_data_ready进行处理

sk_data_ready是在创建内核netlink socket时设置的，通过netlink_data_ready最终调用到audit_init中的audit_receive：在audit_init初始化函数中，在创建netlink socket时会指定一个回调函数用于处理收到的数据：`audit_sock = netlink_kernel_create(NETLINK_AUDIT, audit_receive)`。

audit_receive的调用链：

* audit_receive：
    * skb_dequeue
        * audit_receive_skb
            * audit_receive_msg
            * netlink_ack

在audit_receive_msg中会根据收到的消息的类型执行不同的业务逻辑：

* AUDIT_GET：用户态程序查询audit的状态，对应`auditctl -s`命令
* AUDIT_SET：用户态程序设置audit的配置，对应auditctl的设置命令，例如`auditctl -b`设置backlog_limit，`auditctl -e`设置enabled，`auditctl --reset-lost`重置lost
* AUDIT_USER：接收用户态发送的数据，直接写入到审计日志，对应`auditctl -m`命令
* AUDIT_LOGIN：用户和注销时的审计日志
* AUDIT_LIST：列出系统调用监控规则，对应`auditctl -l`命令
* AUDIT_ADD：增加系统调用监控规则，对应`auditctl -a`命令
* AUDIT_DEL：删除系统调用监控规则，对应`auditctl -d`命令

* audit_receive_msg：
    * audit_receive_filter：
        * AUDIT_LIST：向用户态程序发送audit_tsklist、audit_entlist、audit_extlist的数据，然后发送一个空
        * AUDIT_ADD：将用户空间的规则拷贝到audit_entry，然后根据规则类型，加入到不同的链表
        * AUDIT_DEL：根据规则类型，删除不同的链表中的元素

所以，规则的操作也就是操作audit_tsklist、audit_entlist、audit_extlist三个链表，这三个链表的含义分别是：task、entry、exit。

audit_tsklist的调用链：

* sys_fork arch/x86_64/kernel/process.c
    * do_fork kernel/fork.c
        * copy_process
            * audit_alloc kernel/auditsc.c
                * audit_filter_task
                    * audit_filter_rules(audit_tsklist)

audit_entlist的调用链：

* audit_syscall_entry
    * audit_filter_syscall(audit_entlist)

audit_extlist的调用链：

* audit_syscall_exit
    * audit_get_context
        * audit_filter_syscall(audit_extlist)

### 6 总结

从整个流程来说需要了解以下内容：

* 用户态程序和内核态程序通过netlink机制进行交互，且接口与网络套接字的一致，都是采用类似sendto/recvfrom的系统调用，只是里面的family字段不同
* 2.6.6版本的audit只支持系统调用的审计，不支持文件和目录的监控
* 2.6.6版本的audit中没有内核态的kaudit线程，审计日志的发送在系统调用退出阶段，相当于是个同步的发送过程，可以想象，当审计日志数量比较大时，可能会影响系统调用
