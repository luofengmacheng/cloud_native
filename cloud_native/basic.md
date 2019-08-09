## 云原生关键技术

### 0 关键技术

* 微服务(microservice)
* 容器(container)
* 容器编排
* 不可变基础设施
* 声明式API
* DevOps
* 持续交付

### 1 CNCF Lanscape中各层次的技术

网络：Flannel、Calico
存储：CSI、Ceph
容器：Containerd、CRI-O
镜像：Harbor、Docker Registry
容器编排和管理：Kubernetes、CoreDNS、gRPC、etcd
应用打包：Helm
安全：Notary、Kube-bench
服务网格(Service Mesh)：Envoy、Istio
Serverless：Kubeless、knative
Edge：KubeEdge
Api平台：KubeFlow、Argo

### 2 微服务

* API网关
* 服务注册和服务发现
* 负载均衡
* 服务间通信：消息队列、RESTful、RPC(gRPC、Thrift)
* 服务降级：组件出现故障时，隔离故障
* 断路器(熔断器)：当在短时间内发生指定类型的错误，断路器会开启。断路器通常在一定时间后关闭，以便为底层服务提供足够的时间来恢复。
* 舱璧模式：保证服务之间不会相互影响。
* 冗余和容灾

### 3 服务网格(Service Mesh)

服务网格是致力于解决服务间通讯的基础设施层。Service Mesh通常是通过一组亲良机网络代理(Sidecar proxy)，与应用程序部署在同一个pod中实现，且对应用程序透明。

代表组件：Istio

### 4 不可变基础设施

任何基础设施的实例(包括服务器、容器等各种软硬件)一旦创建之后便成为一种只读状态，不可对其进行任何更改。如果需要修改或者升级某些实例，唯一的方式就是创建一批新的实例以替换。

优势：

* 提升发布应用效率
* 没有雪花服务器(每一片雪花都是独一无二的)
* 快速水平扩展
* 简单的回滚和恢复

### 5 DevOps

一组过程、方法和系统的统称，建立开发、测试、运维之间的合作。从而，缩短开发周期，增加部署频率，更可靠地发布。

* Culture 文化
* Automation 自动化 CI/CD
* Measurement 可测量 监控

### 6 IaC(Infrastructure As Code)

基础设施自动化，像管理代码一样管理基础设施。例如，当我们要部署一套环境时，我们会创建很多虚拟机，配置好网络策略，配置好负载均衡。如果所有这些环境的配置。

代表组件：Terraform

### 7 如何实现云原生

* 容器化：微服务架构
* CI/CD
* 编排和应用定义
* 监控
* 服务发现
* 网络策略
* 分布式存储
* 服务通信
* 软件仓库
* 软件分发