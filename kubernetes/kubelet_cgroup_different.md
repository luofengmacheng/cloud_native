## kubernetes排障：kubelet cgroup driver: "systemd" is different from docker cgroup driver: "cgroupfs"

### 1 问题背景

测试环境的k8s集群突然起不来了，查看apiserver所在节点的系统日志，发现如下报错：

```
kubelet: F0825 03:39:04.876379   13808 server.go:273] failed to run Kubelet: failed to create kubelet: misconfiguration: kubelet cgroup driver: "systemd" is different from docker cgroup driver: "cgroupfs"
```

看错误描述是kubelet的cgroup driver与docker的不同导致的。

### 2 cgroup driver

cgroup是操作系统控制资源使用的一种机制，能够限定一组进程使用的CPU、内存等资源。使用cgroup机制时，是通过在`/sys/fs/cgroup`目录中指定这一组进程的pid以及资源的使用上限。

在容器环境下，有两种使用cgroup的方式：一种是直接操作cgroup在操作系统上的文件，另一种是通过一层代理的方式间接使用cgroup，这两种方式就分别对应了cgroup driver中的cgroupfs和systemd。

* cgroupfs：直接与cgroup在操作系统上的文件交互
* systemd：使用systemd提供的接口来管理cgroup

容器运行时和k8s都可以设置cgroup driver，并且两者的配置必须一致，否则就会报上述的错误信息。

通常建议是：如果系统使用systemd管理进程，则使用systemd作为cgroup driver。

### 3 Docker的cgroup driver配置

docker当前使用的cgroup driver可以通过`docker info`命令查看：

![docker info查看cgroup driver](https://github.com/luofengmacheng/cloud_native/blob/master/kubernetes/pics/docker_info.jpg)

有两种修改docker的cgroup driver的方式：

* 直接在`docker.service`的`ExecStart`的启动命令的后面追加参数：--exec-opt native.cgroupdriver=systemd
* 编辑`/etc/docker/daemon.json`文件，增加`exec-opts`参数：`{ "exec-opts": ["native.cgroupdriver=systemd"] }`

需要注意的是：尽量不要在两个地方都设置该参数，如果都设置也必须设置成相同的值。

### 4 k8s的cgroup driver配置

在k8s的组件中，kubelet负责与容器运行时交互，因此，cgroup driver的配置就在kubelet的配置yaml中。

kubelet的`KubeletConfiguration`的yaml中的`cgroupDriver`配置用于指定cgroup driver。

有时候会发现这里设置没用，此时可以尝试删除`/usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf`文件的`ExecStart`中的`$KUBELET_KUBEADM_ARGS`参数。

修改完成后，先执行`systemctl daemon-reload`，然后将docker和kubelet的服务先后重启，再观察系统日志是否有报错。


