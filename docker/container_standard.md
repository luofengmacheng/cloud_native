## 容器相关的标准

### 0 前言

在容器领域，由于相关利益的驱使，导致出现了各种各样的标准和缩写，搞得人晕头转向，这里不会过多介绍容器的历史，而只是解释下一些相关标准和缩写。

### 1 Container

* OCI(Open Container Initiative)：由Linux基金会主导，主要包含容器运行时的规范，Docker将自家使用的libcontainer封装后，改名为runC

### 2 Kubernetes

* CRI(Container Runtime Interface)：由CNCF主导，主要是为了使kubernetes能够对接不同的容器运行时而制定的接口规范

* runC：实际上是一个命令行工具，与操作系统对接，实现容器的创建和管理
* [containerd](https://containerd.io/)：Docker从Docker Daemon拆分出containerd，containerd负责容器的各种操作
* [CRI-O](https://cri-o.io/)：由redhat开源并由社区驱动的专为kubernetes设计的轻量级容器运行时
* [Kata-containers](https://katacontainers.io/)：






















### 参考文档：

* [白话 Kubernetes Runtime](https://aleiwu.com/post/cncf-runtime-landscape/)