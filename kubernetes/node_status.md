## K8S节点状态的维护

### 1 节点状态

节点是K8S集群中的一类重要资源，节点的状态通常可以作为判断集群异常的重要手段。

为了展示节点在各方面的健康程度，在kubectl describe node k8s-master的输出结果中的Conditions部分可以查看k8s-master节点的一些状态数据：

* NetworkUnavailable：网络配置是否正常
* MemoryPressure：是否还有可用内存
* DiskPressure：是否还有可用磁盘
* PIDPressure：节点上是否有太多的进程
* Ready：节点是否健康

每种类型用5个字段描述：

* Status：状态，通常有三种状态：True、False、Unknown
* LastHeartbeatTime：上一次心跳事件，也就是上次上报该状态信息的时间
* LastTransitionTime：上一次状态转换的时间
* Reason：处于当前状态的原因
* Message：状态描述

NetworkUnavailable状态不是实时的，而是相关的网络插件启动后上报一次，表示网络插件正常启动，但是也无法表明当前的网络是否是正常的，而且，插件运行过程中也没办法知道当前网络是否正常，因此，NetworkUnavailabel只是表明插件成功启动过，无法知道当前的网络是否真的良好。

MemoryPerssure、DiskPressure、PIDPressure都可以通过采集当前节点的数据得到：

* MemoryPerssure：采集当前节点的内存信息，
* DiskPressure：采集当前节点的磁盘信息，
* PIDPressure：采集当前节点的进程信息，

这三个状态值都是由kubelet定时向apiserver上报状态而来，而kube-controller-manager会判断一段时间未上报的情况。

kubectl get node展示的状态即为Conditions中的Ready，该状态由多个条件综合计算而来。

上述状态更新的流程：

* kubelet会每隔一段时间（kubelet的--node-status-update-frequency参数，默认为10秒）更新节点状态到apiserver
* NodeController每隔一段时间（kube-controller-manager的--node-monitor-period参数，默认为5秒）检查一次节点状态
* 如果kubelet超过一段时间（kube-controller-manager的--node-monitor-grace-period参数，默认为50秒）未向apiserver上报，NodeController将节点状态标记为Unknown

### 2 Kubelet

kubelet在启动时会先上报一次节点状态。

* fastNodeStatusUpdate（pkg/kubelet/kubelet_node_status.go）
    * originalNode, err := kl.GetNode() 获取当前节点的信息
    * readyIdx, originalNodeReady := nodeutil.GetNodeCondition(&originalNode.Status, v1.NodeReady) 获取当前节点的Condition中的NodeReady，如果为True，则直接返回
    * node, changed := kl.updateNode(ctx, originalNode) 更新当前节点的信息，并返回新的节点信息以及节点信息是否变化的状态，如果节点状态没有变化，则直接返回
    * readyIdx, nodeReady := nodeutil.GetNodeCondition(&node.Status, v1.NodeReady) 获取当前节点的最新信息中的NodeReady状态，如果为False，则直接返回
    * kl.patchNodeStatus(originalNode, node) 此时节点状态发生了由NotReady向Ready的变化
        * nodeutil.PatchNodeStatus 向apiserver上报节点Ready（毕竟节点没办法自己上报NotReady）
    * kl.syncNodeStatus() 如果上面向apiserver上报节点Ready失败了，则调用syncNodeStatus()
        * kl.registerWithAPIServer() 向apiserver注册节点，如果已经注册过了则直接返回
        * kl.updateNodeStatus(ctx) 更新节点状态
            * kl.tryUpdateNodeStatus(ctx, i)
                * util.FromApiserverCache(&opts) 将GET操作的选项中的ResourceVersion设置为0
                * originalNode, err := kl.heartbeatClient.CoreV1().Nodes().Get(ctx, string(kl.nodeName), opts) 调用apiserver接口获取节点信息，为了减少apiserver访问etcd带来的性能影响，这里调用时apiserver会从缓存中获取，而不是读取etcd
                * updatedNode, err := kl.patchNodeStatus(originalNode, node) 如果检测出apiserver返回的节点信息和当前获取到的节点信息有变化，或者距离上次节点信息上报超过`--node-status-update-frequency`参数，则向apiserver更新节点状态

kubelet启动时为了尽可能快地将节点状态上报给apiserver，每隔100毫秒执行一次fastNodeStatusUpdate函数，有两种情况结束执行：

* 成功地将Ready状态上报给apiserver
* 启动后超过2分钟状态还未更新

在kubelet启动后，会每隔`--node-status-update-frequency`时间上报一次：

``` golang
// pkg/kubelet/kubelet.go
// func (kl *Kubelet) Run(updates <-chan kubetypes.PodUpdate)
go wait.JitterUntil(kl.syncNodeStatus, kl.nodeStatusUpdateFrequency, 0.04, true, wait.NeverStop)
```

### 3 NodeLifecycleController

节点控制器在K8S中是NodeLifecycleController（节点生命周期控制器）。

kube-controller-manager在创建NodeLifecycleController时会传递三个参数：

* NodeMonitorPeriod：检查节点状态的周期
* NodeStartupGracePeriod：节点启动的缓冲期，当节点创建或者节点重启时会预留一段时间
* NodeMonitorGracePeriod：检查节点状态的缓冲期

NodeLifecycleController在启动时会创建一个定时任务的协程：

``` golang
// pkg/controller/nodelifecycle/node_lifecycle_controller.go
// func (nc *Controller) Run(ctx context.Context)
go wait.UntilWithContext(ctx, func(ctx context.Context) {
		if err := nc.monitorNodeHealth(ctx); err != nil {
			logger.Error(err, "Error monitoring node health")
		}
	}, nc.nodeMonitorPeriod)
```

monitorNodeHealth的调用流程如下：

* monitorNodeHealth
    * nodes, err := nc.nodeLister.List(labels.Everything()) 获取所有的节点
    * added, deleted, newZoneRepresentatives := nc.classifyNodes(nodes) 将节点分为三类：新增的、删除的、no zone states？
    * 分别对上述的三种类别的节点进行处理
    * workqueue.ParallelizeUntil(ctx, nc.nodeUpdateWorkerSize, len(nodes), updateNodeFunc) 并发执行updateNodeFunc函数，updateNodeFunc的调用流程如下：
        * node := nodes[piece].DeepCopy() 函数的参数是个索引piece，然后从获取到的所有节点中根据索引piece取出节点
        * _, observedReadyCondition, currentReadyCondition, err = nc.tryUpdateNodeHealth(ctx, node) 尝试更新节点的健康状态
            * _, currentReadyCondition := controllerutil.GetNodeCondition(&node.Status, v1.NodeReady) 获取节点的Condition中的NodeReady
            * 如果当前节点的的ReadyCondition为空，说明kubelet从未上报过节点状态，则将gracePeriod设置为NodeStartupGracePeriod，否则将gracePeriod设置为NodeMonitorGracePeriod
            * observedLease, _ := nc.leaseLister.Leases(v1.NamespaceNodeLease).Get(node.Name) 获取节点的lease
            * 如果距离上次探测时间超过gracePeriod，说明节点状态更新已经超过缓冲期，则执行以下逻辑：
                * 遍历所有的Condition类型(此处不考虑NodeNetworkUnavailable)
                * 如果没有某个类型的Condition，则将Condition中的Reason设置为NodeStatusNeverUpdated
                * 如果存在某个类型的Condition，则将Condition中的Reason设置为NodeStatusUnknown
                * 如果Condition发生变化，则调用nc.kubeClient.CoreV1().Nodes().UpdateStatus(ctx, node, metav1.UpdateOptions{})更新Node的Status

### 4 总结

kubelet和kube-controller-manager更新Conditions的流程与前面说的基本一致：

* kubelet会每隔一段时间（kubelet的--node-status-update-frequency参数，默认为10秒）更新节点状态到apiserver
* NodeController每隔一段时间（kube-controller-manager的--node-monitor-period参数，默认为5秒）检查一次节点状态
* 如果kubelet超过一段时间（kube-controller-manager的--node-monitor-grace-period参数，默认为50秒）未向apiserver上报，NodeController将节点的Condition中的Ready状态修改为False
