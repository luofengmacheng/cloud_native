## 使用Fanotify拦截文件操作

### 1 Fanotify的阻断能力

前文已经说过，Fanotify相比Inotify，比较大的优势是获取的数据多以及Fanotify的阻断能力。

阻断能力，顾名思义，就是在用户操作文件时，如果判断出文件是有问题的，可以不让用户操作。

要使用Fanotify的阻断能力，需要进行以下几个操作：

* fanotify_init创建文件描述符时需要设置FAN_CLASS_PRE_CONTENT或者FAN_CLASS_CONTENT标志位
* fanotify_mark添加文件路劲时设置FAN_OPEN_PERM或者FAN_ACCESS_PERM，分别拦截open和read系统调用，如果要拦截写操作，可以使用FAN_OPEN_PERM
* 通过select+read接收事件时，如果事件类型是FAN_OPEN_PERM或者FAN_ACCESS_PERM时，则使用write写入是否允许继续操作

### 2 Fanotify拦截文件操作的实现

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
        printf("usage: ./perm path\n");
        return -1;
    }

    // 初始化fanotify的描述符，后续的操作都是通过该描述符
    // 此时，flags设置为FAN_CLASS_PRE_CONTENT
    fan_fd = fanotify_init(FAN_CLASS_PRE_CONTENT, O_RDONLY);
    if (fan_fd < 0) {
        perror("fanotify_init");
        return -1;
    }

    // 增加监控路径和事件类型，这里表明需要对open和read进行阻断
    if (fanotify_mark(fan_fd, FAN_MARK_ADD, FAN_OPEN_PERM|FAN_ACCESS_PERM, AT_FDCWD, argv[1]) < 0) {
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

        struct fanotify_response resp;
        // 遍历返回的变更事件
        for (; FAN_EVENT_OK(metadata, len);
            metadata = FAN_EVENT_NEXT(metadata, len)) {
            printf("fd=%d pid=%d ", metadata->fd, metadata->pid);
            if (metadata->mask & FAN_ACCESS) {
                printf("event=read ");
            } else if (metadata->mask & FAN_MODIFY) {
                printf("event=write ");
            } else if(metadata->mask & FAN_OPEN_PERM) {
                printf("event=open_perm response=allow ");
                // 收到open调用，返回允许
                resp.fd = metadata->fd;
                resp.response = FAN_ALLOW;
                if(write(fan_fd, (char*)&resp, sizeof(resp)) < 0) {
                    perror("write reponse");
                    close(metadata->fd);
                    continue;
                }
            } else if(metadata->mask & FAN_ACCESS_PERM) {
                printf("event=access_perm response=deny ");
                // 收到read调用，返回禁止
                resp.fd = metadata->fd;
                resp.response = FAN_DENY;
                if(write(fan_fd, (char*)&resp, sizeof(resp)) < 0) {
                    perror("write reponse");
                    close(metadata->fd);
                    continue;
                }
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
            close(metadata->fd);
        }
    }
    close(fan_fd);

    return 0;
}
```
