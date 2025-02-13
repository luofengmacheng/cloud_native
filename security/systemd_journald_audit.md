## systemd-journald日志服务：systemd-journald-audit.socket

### 1 systemd-journald日志服务

CentOS在启动后，会创建两个进程：1号进程systemd，2号进程kthreadd，分别负责管理用户态进程和内核进程。而Ubuntu中，1号进程是init进程，systemd则是作为用户态的服务启动。

systemd-journald是由systemd管理的系统日志服务，可以收集日志并将日志保存在二进制文件中，然后通过journalctl命令查看日志。

systemd-journald的配置文件位于/etc/systemd/journald.conf，其中列出的配置是当前的默认配置，如果需要调整可以根据`man journald.conf`的说明进行调整。

配置解释如下：

* Storage=auto：日志保存的地方，可以取以下值：volatile(保存在内存)、persistent(保存在磁盘)、auto(根据/var/log/journal目录是否存在决定是保存在内存还是保存在磁盘)、none(接收到日志后可以转发，然后丢弃，并不存储)
* Compress=yes：保存日志文件时是否压缩
* SyncIntervalSec=5m：日志从内存同步到磁盘的时间间隔
* RateLimitIntervalSec=30s、RateLimitBurst=10000：控制单个服务可以产生的日志速率
* SystemMaxUse=、SystemKeepFree=、SystemMaxFileSize=、SystemMaxFiles=100：磁盘上的日志文件的控制参数
* RuntimeMaxUse=、RuntimeKeepFree=、RuntimeMaxFileSzie=、RuntimeMaxFiles=100：内存中的日志文件的控制参数
* MaxRetentionSec：日志文件保存的最长时间
* MaxFileSec=1month：日志文件轮转的最长时间
* ForwardToSyslog=yes：收到日志后转发给syslog，一般就是指rsyslog
* MaxLevelSyslog=debug：转发给syslog的日志的最大级别
* LineMax=48K：单行日志的最大长度
* ReadKMsg=yes：控制是否读取内核产生的日志/dev/kmsg

这些配置都是对日志文件的一些参数配置，防止日志文件占用太多空间，但是，对于用户来说，比较关心两个问题：

* 既然systemd-journald负责接收日志，那用户自己开发的程序是否也可以发送给systemd-journald？如果可以的话，如何配置或者对接呢？
* 日志文件具体保存在哪里？如何查看日志文件？

### 2 systemd-journald和systemd-journald-audit.socket的关系

systemd-journald本身是一个日志接收和处理的模块，那么它的日志从哪里来呢？

systemd-journald的作用是收集系统启动阶段的日志以及服务在启动和运行中的日志，因此，如果服务是用systemd管理的，打印到标准输出的信息就会作为日志被systemd-journal获取到。

也就是说，systemd-journal的功能是接收系统内部的日志以及服务的输出的日志，而业务程序自己的日志通常不需要发送给systemd-journald进行管理，如果希望被系统获取到，在后续用于排错，可以使用`syslog`(`man 3 syslog`)将日志输出到系统日志中。

使用systemctl查看systemd-journald服务状态时会发现其中有个TriggeredBy字段：

![systemctl status systemd-journald](https://github.com/luofengmacheng/cloud_native/blob/master/security/pics/systemctl_status_systemd_journald.jpg)

该字段是表明一种依赖关系。

systemd-journald.service中有Unit和Service两个部分的配置，其中，Unit通常是一些元数据描述信息，例如Description、Documentation等，Service则是具体的服务启动的方式以及服务的一些相关配置。

这里重点关注systemd-journald.service的Unit中的Requires、After、Before字段（man systemd.unit）：

* Requires=systemd-journald.socket：Requires描述一种强依赖关系，也就是必须先启动Requires才能启动当前服务
* After=systemd-journald.socket systemd-journald-dev-log.socket systemd-journald-audit.socket syslog.socket：当前服务必须晚于After启动
* Before=sysinit.target：当前服务必须早于Before启动

因此，需要先启动systemd-journald-audit.socket才能启动systemd-journald.service。

对于Service则主要关注（man systemd.service）：

* ExecStart=/lib/systemd/systemd-journald：指定如何启动systemd-journald服务的命令
* Restart=always：自动重启
* Sockets=ssytemd-journald.socket systemd-journald-dev-log.socket systemd-journald-audit.socket：服务启动时从哪些单元继承套接字文件描述符的名称
* Type=notify：启动完成后会向systemd发送一条就绪通知

总的来说，根据systemd-journald.service的配置只能看出该服务依赖systemd-journald-audit.socket，那么systemd-journald-audit.socket又是什么呢？

systemd-journald-audit.socket中的Unit和Socket的字段解释：

Unit：

* Before=sockets.target：当前服务必须比sockets.target早启动，sockets.target作为需要使用socket的服务的依赖起始点
* ConditionSecurity=audit：检查audit功能是否开启
* ConditionCapability=CAP_AUDIT_READ：检查是否拥有读取audit的权限

Socket：

* Service=systemd-journald.service：
* ListenNetlink=audit 1：创建一个套接字去监听netlink，此处表明要接收audit日志

下面用两句话描述systemd-journald-audit.socket服务的作用：

* 该服务启动时需要先检查audit服务的可用性以及自身是否有权限读取audit
* 该服务需要创建socket使用netlink机制去接收audit日志

那么，systemd-journald-audit.socket服务在接收到audit日志后会将日志保存起来。

### 3 systemd-journald保存日志的方式

前面systemd-journald的配置可以看出，日志在保存时可以保存在内存，也可以保存在磁盘，保存在内存中的日志，当机器重启就没了，但是性能会比较好，保存在磁盘中的日志，重启还在，但是性能会差一些。

* 如果`Storage`设置为`volatile`，日志会保存在内存，此时会将日志保存在`/run/systemd/journal`目录，该目录通常挂载的是tmpfs，也就是保存到内存中
* 如果`Storage`设置为`persistent`，日志会保存在磁盘中，此时会主动创建`/var/log/journal`目录，并将日志定期同步到该目录
* 如果`Storage`设置为`auto`，不会主动创建`/var/log/journal`目录，因此，如果`/var/log/journal`目录不存在，此时跟`volatile`模式一样，如果`/var/log/journal`目录存在，此时跟`persistent`模式一样

### 4 journalctl命令的使用

为了提高日志存储的效率，日志保存的格式肯定会采用二进制的方式，因此，无法直接打开日志文件查看，需要使用`journalctl`命令查看。

journalctl常用的查看日志的命令：

* `journalctl`：查看所有日志，如果日志量不多，通常可以查看到从系统启动开始的日志
* `journalctl -u systemd-journald`：查看systemd-journald服务的日志
* `journalctl -g "success=\w+"`：通过正则过滤日志
* `journalctl -f`：与tail -f一样
* `journalctl -n 10`：与tail -n 10一样
* `journalctl -r`：逆序查看日志，可以查看最新的日志

journalctl还有一些管理命令：

* `journalctl --disk-usage`：可以查看日志在磁盘上的占用空间
* `journalctl --vacuum-size=10485760`：将日志的磁盘空间占用减少到某个值一下，此处的单位为字节，这里就是减少到10MB以下
* `journalctl --vacuum-time=1d`：删除1天以前的数据
* `journalctl --verify`：对日志进行一致性检查
* `journalctl --sync`：将未写入的日志同步到磁盘
* `journalctl --relinquish-var`：不将日志写到磁盘，只写到/run
* `journalctl --rotate`：手动进行日志的轮转
* `journalctl --flush`：将/run中的日志刷到/var

### 5 journald的代码调用图

![journald的代码调用图](https://github.com/luofengmacheng/cloud_native/blob/master/security/pics/journald_source_flow.jpg)
