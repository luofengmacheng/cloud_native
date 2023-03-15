## ss(socket statistics)命令实现

### 1 名词解释

* netstat：network statistics，网络统计信息，通过解析/proc/net/tcp展示网络连接信息
* ss：socket statistics，使用netlink展示网络连接信息
* netlink：netlink socket，用户态空间和内核态空间通信的socket
* rtnetlink：routing netlink，用于实现路由和IP，linux实现中是netlink的子集，创建socket时，family设置为AF_NETLINK，protocol设置为NETLINK_ROUTE
* libnetlink：操作netlink的library

### 2 通过netlink获取连接信息

程序开始时，会通过解析选项，将选项变成下面的struct filter和一些变量的值。

``` c++
struct filter {
    int dbs; // 过滤协议，例如，TCP、UDP、UNIX
    int states; // 过滤连接状态，例如，SS_LISTEN、SS_CLOSE
    uint64_t families; // 过滤协议族
    struct ssfilter *f;
    bool kill;
};
```

ss命令的一些选项就是设置该过滤器：

* -t选项就是设置dbs为TCP_DB
* -4 -6就是设置families协议族

当处理好选项后就是执行数据获取，默认的行为就是通过netlink的方式获取数据：

tcp_show -> inet_show_netlink

``` c++
static int inet_show_netlink(struct filter *f, FILE *dump_fp, int protocol)
{
    int err = 0;
    struct rtnl_handle rth, rth2;
    int family = PF_INET;
    struct inet_diag_arg arg = { .f = f, .protocol = protocol };
 
    if (rtnl_open_byproto(&rth, 0, NETLINK_SOCK_DIAG))
        return -1;
 
    if (f->kill) {
        if (rtnl_open_byproto(&rth2, 0, NETLINK_SOCK_DIAG)) {
            rtnl_close(&rth);
            return -1;
        }
        arg.rth = &rth2;
    }
 
    rth.dump = MAGIC_SEQ;
    rth.dump_fp = dump_fp;
    if (preferred_family == PF_INET6)
        family = PF_INET6;
 
again:
    if ((err = sockdiag_send(family, rth.fd, protocol, f)))
        goto Exit;
 
    if ((err = rtnl_dump_filter(&rth, show_one_inet_sock, &arg))) {
        if (family != PF_UNSPEC) {
            family = PF_UNSPEC;
            goto again;
        }
        goto Exit;
    }
    if (family == PF_INET && preferred_family != PF_INET) {
        family = PF_INET6;
        goto again;
    }
 
Exit:
    rtnl_close(&rth);
    if (arg.rth)
        rtnl_close(arg.rth);
    return err;
}
```

* 首先调用rtnl_open_byproto创建套接字，相当于执行socket(AF_NETLINK, SOCK_RAW, NETLINK_INET_DIAG))
* sockdiag_send向内核发送请求
* rtnl_dump_filter对内核返回的数据进行过滤和解析

这里面涉及的几个数据结构：

``` c++
struct sockaddr_nl { // netlink的socket地址结构，对应socketaddr_in
    __kernel_sa_family_t nl_family; /* 设置为AF_NETLINK */
    unsigned short nl_pad;          /* zero*/
    __u32 nl_pid;                   /* port ID*/
    __u32 nl_groups;                /* multicast groups mask */
};
 
struct nlmsghdr {
    __u32 nlmsg_len;   /* 整个消息的长度，sizeof(nlmsghdr)+sizeof(inet_diag_req) */
    __u16 nlmsg_type;  /* 消息的类型，NLMSG_DONE(本次的消息的最后一个报文)，NLM_F_REQUEST(请求消息)，NLM_F_MULTI(多个报文中的一个) */
    __u16 nlmsg_flags; /* Additional flags */
    __u32 nlmsg_seq;   /* Sequence number */
    __u32 nlmsg_pid;   /* Sending process port ID */
};
 
struct inet_diag_req {
    __u8    idiag_family;       /* Family of addresses. */
    __u8    idiag_src_len;
    __u8    idiag_dst_len;
    __u8    idiag_ext;      /* Query extended information */
 
    struct inet_diag_sockid id;
 
    __u32   idiag_states;       /* 要过滤的连接的状态 */
    __u32   idiag_dbs;      /* Tables to dump (NI) */
};
 
struct iovec
  {
    void *iov_base; /* 数据的地址，发送数据时包含struct nlmsghdr和struct inet_diag_req  */
    size_t iov_len; /* 数据的长度  */
  };
 
struct msghdr // 发送的数据
  {
    void *msg_name;     /* 收发数据的地址，设置为&sockaddr_nl  */
    socklen_t msg_namelen;  /* sizeof(sockaddr_nl)  */
 
    struct iovec *msg_iov;  /* 收发的数据，struct iovec的数组地址  */
    size_t msg_iovlen;      /* Number of elements in the vector.  */
 
    void *msg_control;      /* Ancillary data (eg BSD filedesc passing). */
    size_t msg_controllen;  /* Ancillary data buffer length.
                   !! The type should be socklen_t but the
                   definition of the kernel is incompatible
                   with this.  */
 
    int msg_flags;      /* Flags on received message.  */
  };
```

发送数据时，将整个msghdr发送给内核，内核收到数据后，进行解包然后执行对应的逻辑，然后返回数据，此时，用户态程序就需要以类似的逻辑来解析收到的数据：使用recvmsg接收数据，将接收到的数据转换成nlmsghdr，再通过NLMSG_开头的一些宏(NLMSG_OK：正常收到数据；NLMSG_DATA：得到本次收到的报文数据；NLMSG_NEXT：获取下一个报文；NLMSG_DONE：本次的报文处理完毕)对数据进行处理，因此，大体的逻辑就是：

``` c++
        while (1) {
           ssize_t status = 0;
           struct nlmsghdr *h;
           msg = (struct msghdr){reinterpret_cast<void *>(&nladdr),
                                 sizeof(nladdr),
                                 iov,
                                 1,
                                 nullptr,
                                 0,
                                 0};
 
           // 接收数据
           status = recvmsg(fd, &msg, 0);
           if (status < 0) {
               continue;
           }
           if (status == 0) {
               ret = 0;
               break;
           }
           h = (struct nlmsghdr *)buf;
           while (NLMSG_OK(h, status)) {
               int err;
               struct inet_diag_msg *r;
                
 
               // 本批报文处理完毕，退出循环
               if (h->nlmsg_type == NLMSG_DONE) {
                   ret = 0;
                   break;
               }
 
               // 收到错误报文，退出循环
               if (h->nlmsg_type == NLMSG_ERROR) {
                   ret = 0;
                   break;
               }
 
          // 获取报文的数据部分，然后转换成对应的数据类型指针，然后就可以处理其中的字段并输出
               r = (struct inet_diag_msg *)NLMSG_DATA(h);
 
               h = NLMSG_NEXT(h, status);
           }
           if (ret != 1) {
               break;
           }
       }
```

ss命令在实现时是直接使用了rtnl_dump_filter函数对数据进行处理，收到一个报就会调用回调函数show_on_inet_sock：

``` c++
if ((err = rtnl_dump_filter(&rth, show_one_inet_sock, &arg))) {
    if (family != PF_UNSPEC) {
        family = PF_UNSPEC;
        goto again;
    }
    goto Exit;
}
```

在show_one_inet_sock中，主要就是三个调用：

* parse_diag_msg：使用NLMSG_DATA宏获取数据，从数据中提取出对应的字段，然后放到struct sockstat结构体中
* run_ssfilter：如果用户有提供过滤的表达式，则判断是否应该进行后续处理
* inet_show_sock：根据给定的选项打印数据

parse_diag_msg：将收到的数据转成inet_diag_msg后，就可以通过其中的inet_diag_sockid获取连接的信息。

``` c++
struct inet_diag_sockid {
    __be16  idiag_sport;
    __be16  idiag_dport;
    __be32  idiag_src[4];
    __be32  idiag_dst[4];
    __u32   idiag_if;
    __u32   idiag_cookie[2];
#define INET_DIAG_NOCOOKIE (~0U)
};
 
struct inet_diag_msg {
    __u8    idiag_family;
    __u8    idiag_state;
    __u8    idiag_timer;
    __u8    idiag_retrans;
 
    struct inet_diag_sockid id;
 
    __u32   idiag_expires;
    __u32   idiag_rqueue;
    __u32   idiag_wqueue;
    __u32   idiag_uid;
    __u32   idiag_inode;
};
```

inet_show_sock在进行打印输出时会先调用inet_stats_print输出一些基本信息：

* sock_stat_print：打印State、Recv-Q、Send-Q
* inet_addr_print：打印Local Address:Port和Peer Address:Port
* proc_ctx_print：打印进程的SELinux上下文或者进程信息

然后就需要考虑各种选项，在程序开始的部分会根据选项对一些变量赋值，然后在打印时会判断这些变量的值，然后进行相应的输出：

* 如果设置了选项-o、-e、-b，就会设置show_options为1，此时就会打印timer信息
* 如果设置了-e，就会设置show_details为1，此时会输出uid、ino、sk等信息
* 如果设置了-m，就会设置show_mem为1，此时会输出skmem

### 3 通过proc获取连接信息

此时ss命令已经执行结束了，但是回到最开始调用的tcp_show，当inet_show_netlink执行失败，就会采用读取/proc/net/tcp的方式：

``` c++
if (f->families & FAMILY_MASK(AF_INET)) {
    if ((fp = net_tcp_open()) == NULL)
        goto outerr;
 
    setbuffer(fp, buf, bufsize);
    if (generic_record_read(fp, tcp_show_line, f, AF_INET))
        goto outerr;
    fclose(fp);
}
 
if ((f->families & FAMILY_MASK(AF_INET6)) &&
    (fp = net_tcp6_open()) != NULL) {
    setbuffer(fp, buf, bufsize);
    if (generic_record_read(fp, tcp_show_line, f, AF_INET6))
        goto outerr;
    fclose(fp);
}
```

如果是ipv4，就打开/proc/net/tcp，如果是ipv6，就打开/proc/net/tcp6，两个逻辑都是调用generic_record_read和tcp_show_line对文件进行解析，generic_record_read会读取文件的每一行数据，当成功读取到一行数据后就会调用tcp_show_line，因此，tcp_show_line中就是对文件的一行进行解析和打印：

* proc_parse_inet_addr：解析本端和对端地址以及端口号
* 然后使用sccanf读取剩下的字段，例如state、wq、rq等
* 最后打印的逻辑基本跟netlink的方式类似
