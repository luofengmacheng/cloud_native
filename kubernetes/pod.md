## kubernetes中的Namespace和pod

### 1 为什么需要有Namespace和pod?

pod是容器的上一层概念，由于k8s中Namespace也是分层次的概念，因此，一起拿来说明。

Namespace(命名空间)用于对集群进行划分，在逻辑上将集群分成多个小集群，可以根据部署环境或者是项目组对集群进行划分。

容器的最佳实践是，一个容器中只有一个进程，或者是有相互关联的父子进程，如果k8s以容器为单位进行管理，有时候多个进程之间会有强烈的依赖关系，从部署上来说，必须部署在一起，而又不能在一个容器中，因此，k8s以pod这种容器组为单位进行管理。在k8s中，pod又类似虚拟机，每个pod有独立的内部虚拟IP，并且pod中的容器共享相同的网络。

从层次上来说，通过Namespace和pod将整个k8s集群进行了层次划分：

k8s集群 -> Namespace -> pod -> 容器

### 2 k8s中Namespace的操作

k8s部署完成后，会默认创建三个Namespace：default、kube-public、kube-system。其中kube-public和kube-system由k8s自己使用，用户在不指定Namespace时的所有操作都是在default中操作的。如果用户需要在特定的Namespace中操作，需要在命令中指定Namespace：`--namesapce myspace`或者`-n myspace`。

* 查看Namespace：`kubectl get namespace`
* 创建Namespace：`kubectl create namespace myspace`
* 删除Namespace：`kubectl delete namespace myspace`

### 3 k8s中以yaml文件创建pod

有三种方式可以创建pod：

* 直接执行`kubectl run`，根据镜像创建pod
* 使用pod的yaml文件创建
* 使用ReplicationController、ReplicaSet或者Deployment创建

通过`kubectl run`命令创建pod的方式不方便进行版本管理，而且当前的版本(1.13)使用这种方式创建pod时也是默认创建Deployment对象。

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-test
  namespace: love-test
spec:
  containers:
  - image: nginx
    name: nginx1
    ports:
    - containerPort: 80
      protocol: TCP
```

k8s中所有的对象都可以通过yaml格式的资源文件创建，基本所有的资源文件都会包含四个字段：apiVersion(指定api的版本)、kind(指定对象的类型)、metadata(指定资源的元数据，主要是一些属性信息)、spec(指定对象的构成，用于创建对象)。可以通过`kubectl explain pods`查看每个字段的含义，也可以通过`kubectl explain pods.metadata`查看源数据字段中的每个字段的含义。

因此，上面这个yaml文件的含义是：在love-test名字空间中创建pod，pod的名字为nginx-test，pod中的容器的镜像为nginx，容器的名字为nginx1，容器的端口为80。使用以下命令可以将上面的配置变成pod：

```
kubectl create -f nginx_pod.yaml
```

因此，删除pod就可以用：

```
kubectl delete -f nginx_pod.yaml
```

### 4 标签

k8s中标签是一种将对象(不只是pod)进行分组的方式，在创建对象时可以给对象加上标签，一个对象可以打上多个标签，但是，只给对象打上标签是没用的，还需要结合标签选择器才能发挥作用，标签选择器根据标签选择对象。其实，很多地方都会用到标签，只是在使用标签时没有k8s中这么灵活，例如：在创建版本时会给版本打上标签，表明这个版本的版本号，其实还可以给这个版本打上环境的标签，表明这个版本部署时是需要部署到正式环境还是测试环境。

#### 4.1 添加标签

同样的，可以直接通过kubectl对正在运行的对象进行label的操作，例如pod，当然，更好的方式仍然是在yaml文件中添加，例如，可以在`pods.metadata.labels`中设置pod的标签。

``` shell
# 查看pod时展示标签
kubectl get pods --show-labels

# 展示pod时，只展示env标签(还是会展示所有pod)
kubectl get pods -L env
```

#### 4.2 标签选择器

给对象添加标签后，就可以通过标签选择器对特定的某些对象进行操作，这里还是以pod为例。当给pod添加两个标签：`env=production`和`app=frontend`后，可以通过标签选择器进行选择：

``` shell
# 只展示带env标签的pod
kubectl get pods -l env

# 只展示env=production的pod
kubectl get pods -l env=production

# 展示没有env标签的pod
kubectl get pods -l '!env'
```

通过这种方式，我们可以获取到某些特定的需要关注的对象，并对这些对象进行进一步操作。

#### 4.3 使用标签约束od调度

k8s将多台设备变成一个资源池，通常我们不应该手动干预pod的调度，应该让k8s给自动调度pod，但是，在某些特殊的场景，我们可能需要将pod调度到特定的某些工作节点，例如，需要将某些pod调度到带有SSD的工作节点。

```
kubectl label node kubenode1 ssd=true
```

然后在创建pod时，设置节点的选择器：

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-test
  namespace: love-test
spec:
  nodeSelector:
    ssd: "true"
  containers:
  - image: nginx
    name: nginx1
    ports:
    - containerPort: 80
      protocol: TCP
```

那么，在创建pod时就会将该pod调度到ssd标签为true的工作节点。

### 5 小结

本节主要讲解k8s中几个基础的概念及相应的操作：Namespace、Pod、Label：

* Namespace对集群进行逻辑上的划分
* Pod基于容器之上，让k8s可以对容器组进行管理
* Label基于Pod之上，可以对Pod进行分组，便于对Pod进行管理