## shell中exec命令的使用

### 1 exec的简单使用

exec命令的功能就跟execve系统调用类似，即在当前进程中用新的程序替换当前的程序，并不会创建新的进程，并且不会执行老的程序的后续指令。

例如，当执行exec ls时，就是在当前shell中执行ls，并且原来的bash会退出。因此，exec命令通常是作为脚本的最后一行。

对于老的程序退出的问题，只有一种情况例外，那就是文件描述符的操作，当使用exec操作文件描述符进行重定向时，后续的指定不会退出。

### 2 nginx镜像的入口脚本解析

``` shell
#!/bin/sh
# vim:sw=4:ts=4:et

# 如果任何命令的退出状态非0，则立即退出脚本，用于尽早捕获错误
set -e

# 如果NGINX_ENTRYPOINT_QUIET_LOGS变量定义了，并且不为空，表示将日志丢弃，否则将日志输出到标准输出
if [ -z "${NGINX_ENTRYPOINT_QUIET_LOGS:-}" ]; then
	# 将描述符3重定向到标准输出
    exec 3>&1
else
	# 将描述符3重定向到/dev/null
    exec 3>/dev/null
fi

if [ "$1" = "nginx" -o "$1" = "nginx-debug" ]; then
    if /usr/bin/find "/docker-entrypoint.d/" -mindepth 1 -maxdepth 1 -type f -print -quit 2>/dev/null | read v; then
    	# 在描述符3中写入2条日志
        echo >&3 "$0: /docker-entrypoint.d/ is not empty, will attempt to perform configuration"

        echo >&3 "$0: Looking for shell scripts in /docker-entrypoint.d/"
        # 执行/docker-entrypoint.d目录下的脚本
        find "/docker-entrypoint.d/" -follow -type f -print | sort -V | while read -r f; do
            case "$f" in
                *.sh)
                    if [ -x "$f" ]; then
                        echo >&3 "$0: Launching $f";
                        "$f"
                    else
                        # warn on shell scripts without exec bit
                        echo >&3 "$0: Ignoring $f, not executable";
                    fi
                    ;;
                *) echo >&3 "$0: Ignoring $f";;
            esac
        done

        echo >&3 "$0: Configuration complete; ready for start up"
    else
        echo >&3 "$0: No files found in /docker-entrypoint.d/, skipping configuration"
    fi
fi

# $@表示所有参数的列表，每个参数都是一个独立的字符串
# nginx镜像的ENTRYPOINT是/docker-entrypoint.sh，CMD是"nginx" "-g" "daemon off;"
# 此处的命令就是：exec "nginx" "-g" "daemon off;"
exec "$@"
# exit
```