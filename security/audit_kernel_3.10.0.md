## 3.10.0版本的audit审计机制分析

### 1 入口

与2.6.6相比，audit增加了较多的源文件：

* audit_fsnotify.c
* audit_tree.c：
* audit_watch.c：监控inode
* audit.c：audit的主逻辑
* auditfilter.c：过滤？
* auditsc.c：系统调用审计

### 2 kauditd

在系统执行`ps -ef | grep kauditd`会发现有一个`[kauditd]`的内核线程，该线程在audit_receive_msg中创建：

``` C
static int audit_receive_msg(struct sk_buff *skb, struct nlmsghdr *nlh)
{
	u32			seq;
	void			*data;
	int			err;
	struct audit_buffer	*ab;
	u16			msg_type = nlh->nlmsg_type;
	struct audit_sig_info   *sig_data;
	char			*ctx = NULL;
	u32			len;

	err = audit_netlink_ok(skb, msg_type);
	if (err)
		return err;

	/* As soon as there's any sign of userspace auditd,
	 * start kauditd to talk to it */
	if (!kauditd_task) {
		kauditd_task = kthread_run(kauditd_thread, NULL, "kauditd");
		if (IS_ERR(kauditd_task)) {
			err = PTR_ERR(kauditd_task);
			kauditd_task = NULL;
			return err;
		}
	}

    ...
}
```