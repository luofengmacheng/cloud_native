## 问题定位技巧

### 1 Pod状态不是Running

* kubectl logs POD 查看Pod的日志
* kubectl logs -p POD 查看Pod重启前的日志
* kubectl describe pod POD 查看Pod异常的时间

### 2 由于启动命令不正确导致的Pod异常

通过kubectl logs POD查看到标准输出的日志中报错，报错信息表明，给定的command和args不正确，那怎么知道给定镜像的cmd是什么呢？

使用crictl可以查看镜像的信息：crictl inspecti IMG_ID。其中就有镜像的ENTRYPOINT和CMD。

这里就需要知道ENTRYPOINT和CMD，以及他们与k8s中的command和args的关系：

* ENTRYPOINT：不能被docker run中提供的命令覆盖，并且将CMD作为默认参数
* CMD：可以被docker run中提供的命令覆盖，

因此，当使用docker run运行镜像时：

* 如果没有指定COMMAND和ARG，则直接使用ENTRYPOINT CMD运行
* 如果指定了COMMAND，则使用ENTRYPOINT COMMAND运行
* 如果指定了COMMAND和ARG，则使用ENTRYPOINT COMMAND ARG运行

K8S中则提供了command和args，用于覆盖镜像中的命令：

* command：用于覆盖镜像中的ENTRYPOINT
* args：用于覆盖镜像中的CMD

因此：

* 如果没有指定command和args，则默认使用镜像的ENTRYPOINT CMD运行
* 如果指定了command，没有指定args，则使用command运行(不带任何参数)
* 如果没有指定command，指定了args，则使用ENTRYPOINT args运行
* 如果指定了command和args，则使用command args运行

总结一下：

* 使用docker run运行镜像时，使用ENTRYPOINT COMMAND ARG运行
* 使用k8s运行镜像时，command覆盖ENTRYPOINT，args覆盖CMD

