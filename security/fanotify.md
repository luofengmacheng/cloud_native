## Linux Fanotify使用入门

### 1 Fanotify vs Inotify

在实现某些功能时，可能需要获取某个文件执行的操作，一种可能的方案是用Audit的路径监控，但是Audit存在性能和内核稳定性问题，这个时候就可以其他的文件变更检测机制。

inotify可以监控文件被创建、修改和访问的事件，当文件或者目录发生变化时，可以得到变更的事件，但是inotify有个比较大的不足，就是无法得到执行操作的进程Pid。fanotify是另一种文件监控机制，在使用上两者类似，都是先调用init函数初始化句柄，然后调用类似watch的函数添加监控路径，再使用select+read的方式读取出变化的事件，根据事件中给出的参数获取文件的路径、事件类型以及其他需要的数据。与inotify相比，fanotify的优势在于：

* 访问控制：inotify只能监控文件的变化，而fanotify可以在用户操作之前决定是否可以继续操作，相当于可以用于实现阻断的功能
* 监控范围：inotify用于监控文件或者目录的变化，如果要监控目录下所有文件(包括子目录)，就需要自行递归添加路径；fanotify则提供directed、per-mount和global三种监控模式，其中global模式可以用于监控整个文件系统
* 进程信息：inotify只能得到某个文件发生了什么事件；fanotify可以得到对文件操作的进程pid，程序可以从/proc/pid读取进程信息

### 2 Fanotify Demo

``` C++
#include <sys/fanotify.h>
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>
#include <errno.h>
#include <stdlib.h>
#include <string>
#include <string.h>

int main(int argc, char *argv[]) {
    int fan_fd, fd;

    if(argc < 2) {
        printf("usage: ./main path\n");
        return -1;
    }

    // 初始化fanotify的描述符，后续的操作都是通过该描述符
    fan_fd = fanotify_init(FAN_CLASS_NOTIF, O_RDONLY);
    if (fan_fd < 0) {
        perror("fanotify_init");
        return -1;
    }

    // 增加监控路径
    if (fanotify_mark(fan_fd, FAN_MARK_ADD, FAN_ACCESS|FAN_MODIFY|FAN_EVENT_ON_CHILD, AT_FDCWD, argv[1]) < 0) {
        perror("fanotify_mark");
        return -1;
    }

    fd_set rfds;
    while (1) {
        // 使用select+read读取变更的事件
        FD_ZERO(&rfds);
        FD_SET(fan_fd, &rfds);
        struct timeval timeo = {.tv_sec = 0, .tv_usec = 500000};
        char buf[4096] = {0};
        int ret = select(fan_fd + 1, &rfds, NULL, NULL, &timeo);
        if (ret < 0) {
            close(fan_fd);
            fan_fd = -1;
            break;
        } else if (ret == 0) {
            continue;
        }
        struct fanotify_event_metadata* metadata =
            reinterpret_cast<struct fanotify_event_metadata*>(buf);
        ssize_t len = read(fan_fd, buf, sizeof(buf));

        if (len == -1 && errno == EAGAIN) {
            continue;
        }

        if (len == -1) {
            perror("read");
            exit(EXIT_FAILURE);
        }

        // 遍历返回的变更事件
        for (; FAN_EVENT_OK(metadata, len);
            metadata = FAN_EVENT_NEXT(metadata, len)) {
            printf("fd=%d pid=%d ", metadata->fd, metadata->pid);
            // 根据返回的事件中的mask判断事件类型
            if (metadata->mask & FAN_ACCESS) {
                printf("event=read ");
            } else if (metadata->mask & FAN_MODIFY) {
                printf("event=write ");
            } else {
                printf("event=other ");
            }

            // 根据返回事件中的fd读取操作的文件路径
            char path[1024] = {0};
            char flink[1024] = {0};
            sprintf(flink, "/proc/self/fd/%d", metadata->fd);
            if (readlink(flink, path, sizeof(path)) < 0) {
                printf("readlink %s failed: %s\n", flink, strerror(errno));
                close(metadata->fd);
                printf("\n");
                continue;
            }
            printf("path=%s\n", path);

            // 此处不能少，由于该fd是在当前进程中，需要在处理完该事件后关闭
            close(metadata->fd);
        }
    }
    close(fan_fd);

    return 0;
}
```

### 3 Fanotify API

通过上述示例代码发现，使用fanotify监控路径的基本套路是：

* 调用fanotify_init初始化文件描述符
* 调用fanotify_mark对路径进行监控
* 使用select+read读取变更事件
* 使用FAN_EVENT_OK和FAN_EVENT_NEXT宏遍历变更事件
* 读取到变更事件后，根据事件中的mask得到事件类型，通过事件中的pid得到进程信息，通过事件中的fd得到文件路径，处理完成后关闭fd
* 关闭fanotify的文件描述符

因此，使用过程中重点是需要了解fanotify_init和fanotify_mark两个API的参数和含义。

#### 3.1 fanotify_init

``` C
#include <fcntl.h>
#include <sys/fanotify.h>

int fanotify_init(unsigned int flags, unsigned int event_f_flags);
```

fanotify_init函数用于初始化文件描述符，flags参数可以使用两类标志，一类是通知类型(notification classes)，另一类是对fanotify的文件描述符的一些属性设置。

通知类型有三类：

* FAN_CLASS_PRE_CONTENT：当文件被访问和文件内容被`修改之前`收到事件，适用于在文件被修改之前需要执行操作的场景
* FAN_CLASS_CONTENT：当文件被访问和文件内容被`修改之后`收到事件，适用于在文件被修改之后需要执行操作的场景
* FAN_CLASS_NOTIF：默认值，当文件被访问时会收到事件，不能进行访问控制

如果程序只需要获取事件，而不需要进行访问控制，可以用默认值，否则就需要根据使用场景进行选择。另外，根据含义，三类事件也是按照从上到下的顺序接收的。

设置fanotify的文件描述符的属性的标志：

* FAN_CLOEXEC：设置fanotify的文件描述符的close-on-exec属性，类似open中的O_CLOEXEC
* FAN_NONBLOCK：将文件描述符设置为非阻塞
* FAN_UNLIMITED_QUEUE：移除事件队列的长度限制，默认限制是16384个事件，该标志位需要CAP_SYS_ADMIN权限
* FAN_UNLIMITED_MARKS：移除监控点数量的限制，默认限制是8192，该标志位需要CAP_SYS_ADMIN权限

除了上述标志位，后续的内核还提供其他的标志位，可以在事件上上报更多数据：

* FAN_REPORT_TID：kernel>=4.20，事件包含线程ID
* FAN_REPORT_FID：kernel>=5.1，事件文件系统信息，通知类型必须是FAN_CLASS_NOTIF
* FAN_REPORT_DIR_FID：kernel>=5.9，事件包含目录信息
* FAN_REPORT_NAME：kernel>=5.9，事件包含目录名，该标志必须和FAN_REPORT_DIR_FID一起使用
* FAN_REPORT_DFID_NAME：FAN_REPORT_DIR_FID|FAN_REPORT_NAME

flags可以设置通知类型中的一种和属性设置中的若干标志。

event_f_flags用于设置收到的事件中的文件描述符的属性，常用的标志位有：

* O_RDONLY：只读
* O_LARGEFILE：对超过2GB文件的支持
* O_CLOEXEC：设置close-on-exec属性

当然，也可以设置其他标志位：O_WRONLY、O_RDWR、O_APPEND、O_DSYNC、O_NOATIME、O_NONBLOCK、O_SYNC。

#### 3.2 fanotify_mark

``` C
#include <sys/fanotify.h>

int fanotify_mark(int fanotify_fd, unsigned int flags,
                  uint64_t mask, int dirfd, const char *pathname);
```

fanotify_mark函数用于操作监控路径，可以增加监控路径，也可以删除监控路径。

fanotify_mark有5个参数：

* fanotify_fd：fanotify_init返回的fanotify的描述符
* flags：对监控点的操作
* mask：要监控的操作
* dirfd：监控路径的fd
* pathname：监控路径

flags是表明需要对监控点进行何种操作，是增还是减：

* FAN_MARK_ADD：增加监控点
* FAN_MARK_REMOVE：移除监控点
* FAN_MARK_FLUSH：删除所有监控点

flags可以从上述三个操作中选择一个，然后再结合以下标志位设置：

* FAN_MARK_DONT_FOLLOW：如果pathname是软链接，只监控软链接本身，而不是软链接指向的文件
* FAN_MARK_ONLYDIR：只监控目录，如果监控的不是目录，则报错
* FAN_MARK_MOUNT：监控pathname指定的挂载点路径，如果pathname不是挂载点，则监控pathname所在的挂载点，挂载点的所有目录和文件都会监控

除了上述标志位，kernel>=4.20还提供FAN_MARK_FILESYSTEM，表示监控pathname所在的整个文件系统。

mask表明要监控的事件类型：

* FAN_ACCESS：读文件或者目录
* FAN_MODIFY：写文件
* FAN_CLOSE_WRITE：写关闭
* FAN_CLOSE_NOWRITE：读关闭
* FAN_OPEN：打开文件或者目录
* FAN_OPEN_PERM：在请求打开文件或者目录时，创建fanotify文件描述符时需要使用FAN_CLASS_PRE_CONTENT或者FAN_CLASS_CONTENT
* FAN_ACCESS_PERM：在请求读取文件或者目录时，创建fanotify文件描述符时需要使用FAN_CLASS_PRE_CONTENT或者FAN_CLASS_CONTENT
* FAN_ONDIR：目录操作的事件，如果没有该标志位，只有操作文件的事件
* FAN_EVENT_ON_CHILD：为监控目录的一级路径添加事件
* FAN_CLOSE：FAN_CLOSE_WRITE|FAN_CLOSE_NOWRITE，关闭文件

以上是从内核版本2.6.37开始支持的事件类型，也有更高版本支持的事件类型：

* FAN_OPEN_EXEC：kernel>=5.0，执行可以执行文件
* FAN_ATTRIB：kernel>=5.1，属性变更
* FAN_CREATE：kernel>=5.1，在监控目录中创建文件或者目录
* FAN_DELETE：kernel>=5.1，在监控目录中删除文件或者目录
* FAN_DELETE_SELF：kernel>=5.1，删除监控路径
* FAN_MOVED_FROM：kernel>=5.1，从监控目录中移走
* FAN_MOVED_TO：kernel>=5.1，移入监控目录
* FAN_MOVE_SELF：kernel>=5.1，移动监控文件或者目录本身
* FAN_OPEN_EXEC_PERM：kernel>=5.0，在请求执行可执行文件时，创建fanotify文件描述符时需要使用FAN_CLASS_PRE_CONTENT或者FAN_CLASS_CONTENT
* FAN_MOVE：FAN_MOVED_FROM|FAN_MOVED_TO

监控路径则依赖dirfd和pathname两个参数：

* 如果pathname为NULL，则监控dirfd指定的文件系统
* 如果pathname为NULL，dirfd设置为AT_FDCWD，则监控当前工作目录
* 如果pathname为绝对路径，则监控该路径，此时忽略dirfd参数
* 如果pathname为相对路径，dirfd不是AT_FDCWD，则监控dirfd目录下的pathname
* 如果pathname为相对路径，dirfd是AT_FDCWD，则监控当前工作目录下的pathname

#### 3.3 三种监控模式

前面说过，fanotify支持三种监控模式：

* directed：只监控需要监控的文件或者目录，如果是目录，可以添加FAN_EVENT_ON_CHILD标志监控目录下的所有文件，但是不会监控子目录下的文件，per-mount和global模式下，FAN_EVENT_ON_CHILD标志无效
* per-mount：监控挂载点中的所有文件或者目录，增加监控路径时设置FAN_MARK_MOUNT标志
* global：监控整个文件系统，增加监控路径时设置FAN_MARK_FILESYSTEM标志，不过该选项是在4.20内核版本才支持

### 4 容器场景使用的问题

从上述可以看出，fanotify在5.1内核增加了很多能力，因此，在使用时需要考虑内核的版本问题，另外，在容器场景也存在一些问题。

如果是监控主机上的路径，directed结合FAN_EVENT_ON_CHILD标志就只能监控目录以及目录下的文件，per-mount可以监控pathname所在的挂载点中所有文件或者目录的变化，而global模式可以监控整个文件系统。

从功能实现角度来看，如果需要监控某个路径下的文件变化，希望可以只设置该路径就行，也就是，当监控/etc目录时，也可以监控到/etc/xxx/目录下的文件变化。

如果是监控容器中的路径，如果使用directed和global模式，就跟主机上的一样，监控merged路径；如果使用per-mount，由于不能跨命名空间，需要使用setns进入到容器所在的命名空间，再添加监控路径，实现起来会麻烦很多。

在处理文件的变更事件时，通过metadata->fd读取到的文件路径是容器中的路径，metadata->pid是宿主机上的进程Pid，通过/proc/pid/cgroup可以得到容器ID，虽然可以根据容器ID调用docker或者CRI的接口获取merged路径，但是也比较麻烦。

### 5 参考文档

* [监听容器中的文件系统事件](https://www.cnblogs.com/sctb/p/17400454.html)
