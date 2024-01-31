## 2.6.32版本的audit审计机制分析

### 1 audit_filter_rules分析

* audit_syscall_entry
    * audit_filter_syscall
        * audit_filter_rules 对规则进行遍历和判断
            * get_task_cred
            * audit_watch_inode/audit_watch_dev(AUDIT_WATCH) 
            * match_tree_refs(AUDIT_DIR)

* audit_syscall_exit
    * __audit_syscall_exit

### 2 路径监控

#### 2.1 增加监控规则

* audit_receive_msg 接收用户态程序发送的消息
    * audit_receive_filter 消息类型为AUDIT_ADD_RULE/AUDIT_DEL_RULE/AUDIT_LIST_RULES时处理，AUDIT_ADD/AUDIT_DEL/AUDIT_LIST是为了向后兼容
        * audit_rule_to_entry/audit_data_to_entry 将audit规则转换成内部的表示形式audit_rule，audit_rule_to_entry是向后兼容，里面没有处理路径监控的逻辑，而在audit_data_to_entry中会处理AUDIT_WATCH选项
            * audit_to_watch
                * audit_init_watch 初始化audit_watch结构体，设置count和path字段
                * audit_get_watch audit_watch中的count字段加1，并将audit_krule中的watch设置为audit_watch
        * audit_add_rule
            * audit_add_watch 如果是路径监控规则则调用inotify接口进行监控
                * audit_get_nd 获取监控路径的信息，将监控路径的信息保存到规则的watch字段
                * inotify_find_watch 根据监控的目录的inode查看是否已经被监控
                * audit_add_to_parent
                * put_inotify_watch

#### 2.2 处理变更事件


