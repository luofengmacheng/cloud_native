## 使用virtual-kubelet扩展k8s

### 1 何为virtual-kubelet？

kubelet是k8s的agent，负责监听Pod的调度情况，并运行Pod。而virtual-kubelet不是真实跑在宿主机上的，而是一个可以跑在任何地方的进程，该进程向k8s伪装成一个真实的Node，但是实际的操作可以自定义。

也就是说，virtual-kubelet向k8s提供与kubelet兼容的接口，而可以自定义底层的具体实现，通常可以用于不同架构之间的配合使用，例如，virtual-kubelet的底层的具体实现是采用kvm实现，或者用另一种方式实现Pod。

virtual-kubelet的使用场景：

* 对接原有的平台：[Kubernetes Virtual Kubelet with ACI](https://github.com/virtual-kubelet/azure-aci)
* 资源的自动扩容：[UCloud UK8S虚拟节点 让用户不再担心集群没有资源](https://blog.ucloud.cn/archives/4949)

### 2 virtual-kubelet的整体架构

[virtual-kubelet](https://github.com/virtual-kubelet/virtual-kubelet)

整个virtual-kubelet仓库重要的目录是：

* cmd/virtual-kubelet/
    * main.go 主函数，负责自动virtual-kubelet，依旧使用了Cobra实现
    * register.go 注册provider，这里实现了个Mock的provider
    * internal/
        * commands/ 程序命令的定义，
        * provider/
            * provider.go provider的接口定义
            * mock/ mock的provider的实现
* internal/ 额外实现的一些包供内部调用
* node/ kubelet中的一些逻辑

因此，整个仓库的整体调用路径是：

* main.go使用Cobra构建命令行操作，然后调用register.go注册provider，进入事件循环
* node/目录下有PodController的实现，这里面就会调用注册的provider

### 3 基于virtual-kubelet库扩展k8s

使用virtual-kubelet扩展k8s有2种方式：

* 1 直接克隆virtual-kubelet仓库，然后在cmd/virtual-kubelet/internal/provider下面新建一个自己的provider，然后实现provider接口的各种方法，然后在cmd/virtual-kubelet/register.go中注册自己实现的provider即可
* 2 新开一个仓库，在里面使用virtual-kubelet包，同样的，只需要实现provider，在主函数中注册，然后将provider的接口对接到自己的实现就行

如果是自己实现virtual-kubelet，通常会使用方式2。

当前，部分云厂商已经开发了自己的virtual-kubelet用于对接自己的容器平台，例如，微软开发了对接ACI的azure-aci，下面重点来分析azure-aci的实现，自己实现的方式也类似。

virtual-kubelet/azure-aci中的重点目录包含三个：

* client：封装对接ACI的接口
* cmd/virtual-kubelet：主函数，日志和配置处理，注册ACIProvider并启动
* provider：ACIProvider的实现，调用client中的函数实现provider