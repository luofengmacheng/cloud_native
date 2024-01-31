## 使用auparse解析auditd审计日志

### 1 审计日志特点

查看auditd.log的日志，审计日志的格式如下：

``` shell
type=SYSCALL msg=audit(1703148319.954:11680975): arch=c000003e syscall=2 success=yes exit=5 a0=1102430 a1=0 a2=1b6 a3=24 items=1 ppid=7752 pid=7761 auid=0 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts6 ses=7 comm="python" exe="/usr/bin/python2.7" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
type=CWD msg=audit(1703148319.954:11680975):  cwd="/root/"
type=PATH msg=audit(1703148319.954:11680975): item=0 name="/mnt/file_747.txt" inode=101222977 dev=08:03 mode=0100644 ouid=0 ogid=0 rdev=00:00 obj=unconfined_u:object_r:mnt_t:s0 objtype=NORMAL cap_fp=0000000000000000 cap_fi=0000000000000000 cap_fe=0 cap_fver=0
type=PROCTITLE msg=audit(1703148319.954:11680975): proctitle=707974686F6E0066696C655F726561642E7079
```

从外层看审计日志有两个字段：type和msg，type表示类型，意思是当前行的日志的含义，msg就是审计日志的信息。在msg里面又包含很多字段，在这些字段的开始部分的括号中有两个字段，分别表示日志时间戳和序列号：audit(timestamp:serial)，时间戳是毫秒级别，而序列号是一个自增的整数，同一个事件的多行日志的时间戳和序列号是一样的。在序列号后面就是若干字段。

以上面的例子来讲解下日志的内容：

* 第一条日志的类型是SYSCALL，表示这个是系统调用，系统调用号是2(ausyscall 2)，表示open系统调用，success=yes表示调用成功，exit=5表示调用的返回值是5，也就是文件描述符的值，后面的a0~a3表示系统调用的参数，然后就是进程的一些信息，如pid、ppid、uid、comm、exe等
* 第二条日志的类型是CWD，表示这个是当前工作路径，日志的内容就只有输出当前工作路径
* 第三条日志的类型是PATH，表示系统调用过程中访问的路径，这里就是打开的文件的信息，例如，文件名为/mnt/file_747.txt，文件的inode号、mode等
* 最后一条日志的类型是PROCTITLE，表示进程标题，也就是cmd，但是这个是需要解析的，无法直接看出cmd

对于这个事件的日志来说，还有一条日志没有打印出来：`type=EOE msg=audit(1703151485.085:11729997):`，这条日志的类型是EOE(End OF Event)，表示事件的结束，由于这条日志没有内容，因此，auditd在输出到日志文件时过滤了这条日志。

虽然上面将这个事件的几条日志放在一起，但是当时间量比较大时，auditd.log中的日志是不保证顺序的，也就是说，多个事件的日志之间会串。

### 2 使用auparse直接解析audispd转发的日志

当使用audispd的af_unix插件转发审计日志时，可以直接通过connect和read读取审计日志，会发现一次可以读取到多条日志，就有可能出现日志被截断以及乱序的问题，此时，可以通过auparse进行解析。

auparse是libaudit中的解析审计日志的模块，可以很方便的提取出审计日志中的字段：

``` C
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/socket.h>
#include <sys/un.h>
#include <libaudit.h>
#include <auparse.h>

void handle_event(auparse_state_t *au, auparse_cb_event_t cb_event_type,
                  void *user_data) {
    int type = 0;
    auparse_first_record(au);
    do {
        type = auparse_get_type(au);
        printf("type=%d\n", type);

        const char *val = auparse_get_record_text(au);
        printf("msg=%s\n", val);

        auparse_first_field(au);

        do {
            printf("%s=%s (%s)\n", auparse_get_field_name(au), auparse_get_field_str(au), auparse_interpret_field(au));
        } while(auparse_next_field(au) > 0);
    } while(auparse_next_record(au) > 0);
    printf("\n");
}

int main(int argc, char *argv[]) {
    int sockfd;
    struct sockaddr_un serv_addr;
    ssize_t len;
    char buffer[1024];

    if (argc != 2) {
        printf("Usage: %s <socket_path>\n", argv[0]);
        exit(1);
    }

    sockfd = socket(AF_UNIX, SOCK_STREAM, 0);
    if (sockfd < 0) {
        perror("socket");
        exit(1);
    }

    serv_addr.sun_family = AF_UNIX;
    strcpy(serv_addr.sun_path, argv[1]);

    if (connect(sockfd, (struct sockaddr *)&serv_addr, sizeof(serv_addr)) < 0) {
        perror("connect");
        exit(1);
    }

    printf("Connected to the Unix Socket: %s\n", argv[1]);
    auparse_state_t *au = auparse_init(AUSOURCE_FEED, 0);
    if(au == NULL) {
        printf("auparse_init failed");
        exit(1);
    }
    auparse_add_callback(au, handle_event, NULL, NULL);

    while ((len = read(sockfd, buffer, sizeof(buffer) - 1)) > 0) {
        buffer[len] = '\0';
        auparse_feed(au, buffer, len);
    }

    if (len < 0) {
        perror("read");
    }

    close(sockfd);
    return 0;
}
```

在创建socket，然后connect连接到unix socket后，就调用auparse_add_callback注册事件处理的回调函数handle_event，当通过read读取到数据后，就调用auparse_feed将审计日志给到解析器，在回调函数中就可以调用auparse_类函数把数据当做链表进行处理：

* auparse_first_record + auparse_next_record：遍历多条审计日志
* auparse_first_field + auparse_next_field：遍历一条审计日志的多个字段
* auparse_get_type：获取审计日志的类型
* auparse_get_record_text：获取审计日志的完整内容

通过auparse就可以快速方便的实现字段的解析和提取，但是需要注意的是，一旦调用auparse_feed将数据给到auparse，auparse会先放到队列，然后进行解析，因此，如果审计日志量级比较大，有可能放到队列的操作会失败，也就是调用auparse_feed失败，如果想增大日志处理的吞吐量，可以创建一个比较大缓存队列，收到日志后，先放到队列，然后再从队列中取出后调用auparse_feed。
