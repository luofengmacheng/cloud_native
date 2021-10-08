## CRI & OCI

* OCI(Open Container Initiative)：由Linux基金会主导，主要包含容器运行时的规范，Docker将自家使用的libcontainer封装后，改名为runC
* CRI(Container Runtime Interface)：由CNCF主导，主要是为了使kubernetes能够对接不同的容器运行时而制定的接口规范

* runC：实际上是一个命令行工具，与操作系统对接，实现容器的创建和管理
* [containerd](https://containerd.io/)：Docker从Docker Daemon拆分出containerd，containerd负责容器的各种操作
* [CRI-O](https://cri-o.io/)：由redhat开源并由社区驱动的专为kubernetes设计的轻量级容器运行时
* [Kata-containers](https://katacontainers.io/)：由OpenStack基金会管理的容器项目，整合了Intel和Clear Containers和Hyper.sh的runV，主要目标是提供虚拟机级别的安全容器，也就是拥有容器的启动速度以及虚拟机的安全

![docker架构](https://github.com/luofengmacheng/docker_doc/blob/master/kubernetes/pics/cri_oci_docker.png)

docker作为命令行工具，当用户使用docker create创建容器时，docker会将命令发给dockerd守护进程，docker会将请求发给docker-containerd，docker-containerd会启动docker-containerd-shim进程，该进程会使用runC创建进程，runC是一个二进制程序，创建结束后就会退出，而docker-containerd-shim会作为容器的父进程而运行。

![kubernetes架构](https://github.com/luofengmacheng/docker_doc/blob/master/kubernetes/pics/cri_oci_k8s.png)

CRI作为k8s提供给外部容器运行时的接口，可以对接许多容器运行时，在早期，k8s通过内置的dockershim实现对docker的兼容，但是在未来，k8s会移除掉dockershim，而是直接对接外部的容器运行时。当前的主流容器运行时主要是：containerd(从dockerd分离而来)和CRI-O。为了管理容器，接收容器中进程的信号，管理容器中的进程，通常会在容器运行时中提供shim，该shim负责容器中进程的管理。具体实现容器和镜像的功能的规范是OCI，当前的主流方案是runC(由dockerd分离而来)和kata(基于虚拟化方案提供安全容器)。

参考文档：

* [白话 Kubernetes Runtime](https://aleiwu.com/post/cncf-runtime-landscape/)