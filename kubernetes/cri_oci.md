## CRI & OCI

### 1 OCI

OCI(Open Container Initiative)：由Linux基金会主导，主要包含容器镜像规范和容器运行时规范：

* [Image Specification(image-spec)](https://github.com/opencontainers/image-spec)
* [Runtime Specification(runtime-spec)](https://github.com/opencontainers/runtime-spec)
* [runC](https://github.com/opencontainers/runc)

image-spec定义了镜像的格式，镜像的格式有以下几个部分组成：

* [index(可选)](https://github.com/opencontainers/image-spec/blob/main/image-index.md)：由于镜像对于不同平台的实现可能不同，因此，对于不同的平台会提供不同的镜像，在该文件中与manifest最大的不同就是platform，它表明了该镜像对应的平台(包含硬件架构和操作系统)
* [manifest](https://github.com/opencontainers/image-spec/blob/main/manifest.md)：描述了镜像自身以及镜像的各层的sha256，用sha256就可以定位到其他层
* layers：每一层的数据的压缩包
* [config](https://github.com/opencontainers/image-spec/blob/main/config.md)：镜像的配置参数

当使用containerd作为容器运行时，上面的所有文件都在/var/lib/containerd/io.containerd.content.v1.content/blobs/sha256中。

runtime-spec定义了容器的配置、运行环境和生命周期：

* [config](https://github.com/opencontainers/runtime-spec/blob/main/config.md)：生成容器的参数配置
* [runtime](https://github.com/opencontainers/runtime-spec/blob/main/runtime.md)：容器的状态、生命周期以及可以执行的操作

当使用containerd作为容器运行时，在`ctr c info <container_id>`的输出的Spec中可以查看到config文件的内容，另外，使用`ctr oci spec`可以查看到runtime-spec的默认config文件。

runC是一个从docker的libcontainer库封装而来的工具，可以根据OCI规范来创建和管理容器，它直接对接操作系统，因此，也使用runC直接创建容器([RunC简介](https://gohalo.me/post/docker-component-runc-introduce.html))，使用runC创建容器时需要2个部分：runtime-spec中的config文件以及rootfs。

### 2 CRI

CRI(Container Runtime Interface)：由CNCF主导，主要是为了使kubernetes能够对接不同的容器运行时而制定的接口规范

* [containerd](https://containerd.io/)：Docker从Docker Daemon拆分出containerd，containerd负责容器的各种操作
* [CRI-O](https://cri-o.io/)：由redhat开源并由社区驱动的专为kubernetes设计的轻量级容器运行时
* [Kata-containers](https://katacontainers.io/)：由OpenStack基金会管理的容器项目，整合了Intel和Clear Containers和Hyper.sh的runV，主要目标是提供虚拟机级别的安全容器，也就是拥有容器的启动速度以及虚拟机的安全

![docker架构](https://github.com/luofengmacheng/cloud_native/blob/master/kubernetes/pics/cri_oci_docker.png)

docker作为命令行工具，当用户使用docker create创建容器时，docker会将命令发给dockerd守护进程，docker会将请求发给docker-containerd，docker-containerd会启动docker-containerd-shim进程，该进程会使用runC创建进程，runC是一个二进制程序，创建结束后就会退出，而docker-containerd-shim会作为容器的父进程而运行。

![kubernetes架构](https://github.com/luofengmacheng/cloud_native/blob/master/kubernetes/pics/cri_oci_k8s.png)

CRI作为k8s提供给外部容器运行时的接口，可以对接许多容器运行时，在早期，k8s通过内置的dockershim实现对docker的兼容，但是在未来，k8s会移除掉dockershim，而是直接对接外部的容器运行时。当前的主流容器运行时主要是：containerd(从dockerd分离而来)和CRI-O。为了管理容器，接收容器中进程的信号，管理容器中的进程，通常会在容器运行时中提供shim，该shim负责容器中进程的管理。具体实现容器和镜像的功能的规范是OCI，当前的主流方案是runC(由dockerd分离而来)和kata(基于虚拟化方案提供安全容器)。

参考文档：

* [白话 Kubernetes Runtime](https://aleiwu.com/post/cncf-runtime-landscape/)