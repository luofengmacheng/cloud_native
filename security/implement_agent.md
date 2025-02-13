## 如何实现一个agent

### 1 什么是agent?

首先问下kimi：

```
我：什么是agent技术
kimi：Agent技术是指在客户端机器上部署一个Agent进程，该进程作为客户端与服务端之间的中介，负责处理客户端的请求并与之进行通信。Agent通常与客户端位于同一主机上，可以通过本地通信方式（如localhost）进行访问。Agent技术能够解决工程效率问题和架构演进的问题，它通过在客户端部署Agent进程，实现了服务端的部分计算和逻辑处理，从而减轻了服务端的压力，并降低了服务端代码的复杂度。此外，Agent技术还有助于基础组件与业务的解耦，使得基础架构的变更不再依赖于业务方的升级和发布，提高了系统的灵活性和可维护性2。
```

简而言之，一种功能的实现不可能完全由服务端完成，特别是对于分布式系统而言，需要在机器上部署一个程序，该程序会接收服务端的消息，然后执行一定的业务逻辑。

举两个例子：

例1：当电脑上安装一些安全软件时，该软件就相当于是agent，该软件启动后会从服务端（也就是安全软件的后台管理端）下载病毒库进行更新，当执行本地文件扫描时，可以先使用一些本地扫描的技术，还可能将本地文件上传到边缘或者云端进行查杀。

例2：在k8s集群中，每个机器上会安装kubelet和containerd（或者是其他运行时），这里的kubelet就相当于是agent，kubelet启动后会连接apiserver，监听pod调度的变更，然后调用containerd管理容器。

总之，agent通常与服务端（或者称为管理端）一起构成分布式系统，agent会接收服务端发来的任务，利用本地算力进行处理，上报任务结果，实现典型的`客户端-服务端模式`。

### 2 实现agent需要考虑的问题

#### 2.1 agent与服务端的交互方式

#### 2.2 agent的异常自监控

#### 2.3 agent的扩展性