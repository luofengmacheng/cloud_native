## kubernetes起步(minikube简单使用)

### 1 minikube简介

minikube是一个轻量级的kubernetes实现，可以在本地直接创建一个虚拟机并在里面运行kubernetes，因此，可以使用minikube入门kubernetes。

### 2 基本概念和操作

#### 2.1 基本概念

学习kubernetes需要了解以下几个基本的概念：

* Namespace：命名空间，用于逻辑上将整个集群分成若干个小集群，便于实现多租户，也便于将整个集群进行业务划分。
* Master：主server，对整个集群进行调度，接收Node的请求并处理。
* Node：节点server，实际运行容器的节点。
* Pod：kubernetes中创建和部署的基本单位。

#### 2.2 基本操作

minikube安装完成后，可以使用kubectl命令对集群进行操作，可以用以下命令查看集群的状态：

``` shell
# 查看客户端和服务端的版本
# 客户端版本：kubectl自己的版本
# 服务端版本：集群的版本
kubectl version

# 查看集群信息
kubectl cluster-info

# 查看集群的节点，可以从结果中看到master和node的信息
kubectl get nodes

# 查看集群中的pod
kubectl get pods
```

### 3 部署简单的应用并对容器进行操作

`kubectl run kubernetes-bootcamp --image=gcr.io/google-samples/kubernetes-bootcamp:v1 --port=8080`

上面的命令跟`docker run`的有些类似，`kubectl run`其实会创建一个Deployment，其它的选项相当于是Deployment的参数：`--image`指定镜像，`--port`指定端口，可以用`kubectl get deployments`进行查看。

我们也可以使用跟docker命令类似的方式操作pod：

```
# 在pod中执行env命令
kubectl exec $POD_NAME env

# 进入pod
kubectl exec -it $POD_NAME bash

# 查看pod的日志
kubectl logs $POD_NAME
```

kubernetes对于运维中的各种场景提供了完整的解决方案，这里就以下三种场景对kubernetes进行学习。

### 4 服务注册与服务发现

kubernetes中可以通过服务对外提供访问，而提供服务的pod是通过label进行选择的，在创建pod时可以指定label，而在创建服务时可以指定要选择的pod的label。

有4种访问服务的方式：

* ClusterIP(默认)，这种方式只能在集群内部访问该服务
* NodePort，使用NAT技术在选定的pod的节点上用同一个端口路由到该服务，这种方式可以在集群外部访问
* LoadBalancer，创建负载均衡器，并为该服务分配固定的外部IP(类似于Elastic IP)
* ExternalName，使用名字访问服务

使用`kubectl expost`命令对外暴露服务：

```
kubectl expose deployment/kubernetes-bootcamp --type="NodePort" --port 8080

# 查看所有的服务
kubectl get services

# 查看服务的详情
kubectl describe services/kubernetes-bootcamp
```

上述命令以NodePort的方式对外提供服务，本地端口是8080，而服务对应的pod则通过Deployment获取。

此时就可以使用(集群IP:服务端口)访问该服务。

上面这种方式是直接使用Deployment默认创建的label选择pod。

我们可以手动给pod加上label：`kubectl label pod $POD_NAME app=v1`，然后可以通过标签选择pod：`kubectl get pods -l app=v1`。同时，可以删除一个服务：`kubectl delete service $SERVICE_NAME`。

### 5 自动扩缩容

kubernetes使用Deployment管理部署，使用`kubectl get deployments`可以查看当前的Deployment。

在结果中有几列：

* DESIRED 期望提供服务的实例个数
* CURRENT 正常启动的个数
* UP-TO-DATE 更新的副本的个数
* AVAILABLE 对用户可用的副本数

使用`kubectl scale`可以手动进行扩容：

```
kubectl scale deployments/kubernetes-bootcamp --replicas=4
```

将该deployment的副本数设置为4。再次查看deployment时会发现列值都变成了4。同时，通过`kubectl get pods -o wide`会查看到每个pod的IP都不一样。

同样的，使用`kubectl scale`也可以进行缩容，只要指定`--replicas`选项，kubernetes就可以根据设定的实例数，自动选择合适的节点进行启动或者停止pod。

### 6 滚动更新(灰度发布)

当运维负责的业务正在发展中时，新版本可能会很频繁，此时，运维就会频繁对现网进行更新，而更新过程中，进行更新的节点可能暂时不能提供服务，为了不对业务造成影响，运维通常会对更新的节点权重降低或者调0，使得流量只会导入到那些不进行升级的节点，待更新完成并验证OK后，运维才会将权重升高，并查看日志检查此次更新是否成功。这在业界也叫`灰度发布`。

kubernetes原生就支持这种滚动更新的方式：设置执行更新的pod数或者比例，那么，在更新时，kubernetes会按照一定的规则逐步对pod进行更新，同时，在出现问题时，也能够轻松回到上个版本。

kubernetes使用`kubectl set image`更新pod的镜像：

```
kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=jocatalin/kubernetes-bootcamp:v2
```

然后kubernetes就会依次对pod进行更新：停止一个容器，并启动一个容器。

如果升级后出现问题，可以执行`kubectl rollout undo deployments/kubernetes-bootcamp`对deployment进行回滚。

### 7 总结

本文使用minikube演示了kubernetes在服务注册与服务发现、自动扩缩容、滚动更新方面的简单使用，都是通过`kubectl`客户端对集群进行操作，在前面也发现。

### 参考文档

1 [学习Kubernetes基础知识](https://kubernetes.io/zh/docs/tutorials/kubernetes-basics/)