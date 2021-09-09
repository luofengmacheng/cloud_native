## kubernetes中的调度

### 1 调度过程

调度的本来含义是指决定某个任务交给某人来做的过程，kubernetes中的调度是指决定Pod在哪个Node上运行。

k8s的调度分为2个过程：

* 预选：去掉不满足条件的节点
* 优选：对剩下符合条件的节点按照一些策略进行排序，选择最优的节点运行Pod

### 2 亲和性

亲和性是指Pod调度时更`倾向`于调度到满足某些条件的节点。

亲和性按照对象不同分为节点亲和性和Pod亲和性：

* 节点亲和性：Pod更倾向于调度到某类节点上运行
* Pod亲和性：Pod更倾向于调度到跟某类Pod运行到同一个节点

亲和性按照满足条件的程度又分为软亲和性和硬亲和性：

* 软亲和性：如果有满足条件的节点，则在这些节点上运行如果没有，也可以调度到不满足条件的节点上运行
* 硬亲和性：必须调度到满足条件的节点上运行

亲和性的配置位于`Pod.spec.affinity`，节点亲和性位于`Pod.spec.affinity.nodeAffinity`，pod亲和性位于`Pod.spec.affinity.podAffinity`。

### 3 污点和容忍度

#### 3.1 污点和容忍度的含义

节点亲和性是指Pod必须或者可以被调度到带有某些标签的节点，是一种节点`吸引`Pod的能力。而污点则是节点`排斥`Pod的能力。

`污点`：节点拥有的属性，可以给节点打上污点，当Pod调度时就不会被调度到带有污点的节点。

`容忍`：创建Pod时指定的容忍度，当Pod调度时可以将Pod调度到带有这些污点的节点，当然也可以不调度到带有这些污点的节点。

污点由三部分组成：`key=value:effect`，其中value可以为空，effect描述污点的作用：

* NoSchedule：k8s不会将Pod调度到有该污点的节点
* PreferNoSchedule：k8s尽量避免将Pod调度到有该污点的节点(当节点资源不足时，也可以被调度)
* NoExecute：k8s不会将Pod调度到有该污点的节点，同时会将节点上已经存在的Pod驱逐

#### 3.2 如何设置

污点的设置：

```
# 设置污点
kubectl taint nodes node-name key=value:effect

# 查看污点
kubectl describe pod pod-name

# 删除污点
kubectl taint nodes node-name key:effect-
```

容忍度的设置：

Pod.spec.tolerations

```
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
  tolerationSeconds: 3600
- key: "key2"
  operator: "Exists"
  effect: "NoSchedule"
```

注意：

* key、value、effect需要与节点上设置的污点一致
* tolerationSeconds表示当污点设置为NoExecute时，Pod被驱逐前可以继续保留运行的时间

### 4 固定节点调度

在某些场景下，可能需要将Pod调度到某些固定的节点上：

* 节点托管：用户将服务器托管给云平台，这些服务器只能用户自己使用，云平台给节点打上标签，用户在提交yaml时在其中配置好节点选择标签
* 集群分区：对单个集群分区，分别打上标签

因此，k8s支持两种指定节点的方式：

`Pod.spec.nodeName`：指定节点名称，Pod就会只运行在该节点上。

`Pod.spec.nodeSelector`：指定节点带有的标签，Pod就只会运行在带有这些标签的节点上。

### 5 节点驱逐

当需要对节点进行维护时，需要让k8s不将Pod调度到该节点，k8s提供三种驱逐节点的操作：

* cordon
* drain
* delete

#### 5.1 cordon

`kubectl cordon NodeName`

将节点设置为SchedulingDisabled，后续新创建的Pod不会再调度到该节点，原来跑在上面的Pod仍可以对外提供服务。可以使用uncordon恢复调度。

#### 5.2 drain

`kubectl drain NodeName`

将节点设置为SchedulingDisabled，后续新创建的Pod不会再调度到该节点，并且，原来跑在上面的Pod会被优雅终止。可以使用uncordon恢复调度。

#### 5.3 delete

`kubectl delete node NodeName`

驱逐节点上的Pod，在其他节点上重建，然后将该节点从集群中删除。如果需要重新加入集群，需要将kubelet进程重启。