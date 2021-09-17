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

### 4 Microsoft的azure-aci

virtual-kubelet/azure-aci的目录结构如下：

* charts/：helm打包生成的压缩包
* client/：封装对接ACI的接口
    * aci/：对ACI容器组的接口封装
    * api/：封装ACI接口时使用的一些功能函数
    * network/：封装ACI的subnet的接口
    * resourcegroups/：封装ACI的资源组的接口
* cmd/virtual-kubelet/main.go：主函数，日志和配置处理，注册ACIProvider并启动
* helm/：helm的配置
* provider/：ACIProvider的实现，调用client中的函数实现provider

main.go中就是一些环境配置和启动函数：

``` golang
// 获取环境变量中的配置
// 例如，节点名、kubeconfig文件、污点
o, err := opts.FromEnv()
if err != nil {
    log.G(ctx).Fatal(err)
}
o.Provider = "azure"
o.Version = strings.Join([]string{k8sVersion, "vk-azure-aci", buildVers, "-")
o.PodSyncWorkers = numberOfWork

// 初始化node节点
node, err := cli.New(ctx,
    cli.WithBaseOpts(o),
    cli.WithCLIVersion(buildVersion, buildTime),
    cli.WithProvider("azure", func(cfg provider.InitConfig) (proviProvider, error) {
        return azprovider.NewACIProvider(cfg.ConfigPath, ResourceManager, cfg.NodeName, cfg.OperatingSystem, InternalIP, cfg.DaemonPort, cfg.KubeClusterDomain)
    }),
    cli.WithPersistentFlags(logConfig.FlagSet()),
    cli.WithPersistentPreRunCallback(func() error {
        return logruscli.Configure(logConfig, logger)
    }),
    cli.WithPersistentFlags(traceConfig.FlagSet()),
    cli.WithPersistentPreRunCallback(func() error {
        return opencensuscli.Configure(ctx, &traceConfig, o)
    }),

if err != nil {
    log.G(ctx).Fatal(err)

if err := node.Run(ctx); err != nil {
    log.G(ctx).Fatal(err)
}
```

上面的核心代码就是WithProvider()，该函数有两个参数，一个是provider的名称，另一个是provider的初始化函数，这里传的初始化函数就是创建ACIProvider：azprovider.NewACIProvider()。

``` golang
func WithProvider(name string, f provider.InitFunc) Option {
    return func(c *Command) {
        if c.s == nil {
            c.s = provider.NewStore()
        }
        c.s.Register(name, f)
    }
}
```

这个地方需要注意的是初始化函数的参数：cfg provider.InitConfig：

``` golang
type InitConfig struct {
    ConfigPath        string
    NodeName          string // 注册到k8s到节点名称
    OperatingSystem   string // 节点的操作系统
    InternalIP        string // 节点的IP
    DaemonPort        int32 // 节点的端口
    KubeClusterDomain string // k8s集群域名
    ResourceManager   *manager.ResourceManager
}
```

provider.InitConfig里面大部分都是VK节点向k8s集群声明自己的一些信息。这些信息是通过初始化函数直接给到provider到初始化函数的，那么这些参数从哪里获得呢？

第一种方式，前面调用了`opts.FromEnv()`，该函数会从环境变量中获取一些信息，但是这个只有很少量的信息：节点名、端口、kubconfig、污点。

第二种方式，在执行azure-aci时传一些命令行参数，通过查看`cli.New()`函数的实现发现，该函数返回的实际上是个Command，该类型的cmd是个cobra.Command，cmd在创建命令时调用了`installFlags(cmd.Flags(), o)`，该函数会添加很多命令行选项，其中包含常见的的cluster-domain、nodename、provider等。其实，直接编译执行也能发现该程序等命令行参数。

这些命令行参数里面，比较重要的是：

* disable-taint：关闭污点，如果VK节点需要调度Pod，就需要启用该选项
* nodename：节点名称
* cluster-domain：集群域名，连接k8s集群时使用
* no-verify-clients：当请求访问VK节点时，不验证客户端证书

接下来的重点就是`azprovider.NewACIProvider()`的实现，从上面的目录结构也可以看出，provider是对ACIProvider对provider接口的实现，client是对ACI接口的封装，在实现ACIProvider过程中调用client进行对接：

```
provider -> client -> ACI
```

`NewACIProvider()`函数在provider/aci.go中实现，其中的核心逻辑是：

``` golang
// 创建ACI客户端
p.aciClient, err = aci.NewClient(azAuth, p.extraUserAgent, p.retryConfig)
if err != nil {
    return nil, err
}

// 设置节点容量
if err := p.setupCapacity(context.TODO()); err != nil {
    return nil, err
}
```

接下来就是接口实现，kubelet最本质的工作就是监听Pod的状态变更，然后执行相应的动作，因此，当然是需要实现Pod的相关操作。

下面是VK所有的接口：

``` golang
// pod控制器调用的接口，用于管理pod的生命周期
type PodLifecycleHandler interface {
    // 创建Pod
    CreatePod(ctx context.Context, pod *corev1.Pod) error

    // 更新Pod
    UpdatePod(ctx context.Context, pod *corev1.Pod) error

    // 删除Pod
    DeletePod(ctx context.Context, pod *corev1.Pod) error

    // 查询单个Pod，返回的Pod有可能被多个goroutine并发访问，
    // 因此，最好使用DeepCopy深拷贝
    GetPod(ctx context.Context, namespace, name string) (*corev1.Pod, error)

    // 查询单个Pod对应的状态，同样的，需要使用DeepCopy
    GetPodStatus(ctx context.Context, namespace, name string) (*corev1.PodStatus, error)

    // 查询provider上运行的所有Pod，同样的，需要使用DeepCopy
    GetPods(context.Context) ([]*corev1.Pod, error)
}

// 下面是必须要实现的函数，除了Pod，还包含其他的相关函数
type Provider interface {
    node.PodLifecycleHandler

    // 返回某个容器的日志(kubectl logs)
    GetContainerLogs(ctx context.Context, namespace, podName, containerName string, opts api.ContainerLogOpts) (io.ReadCloser, error)

    // 在容器中执行命令(kubectl exec)
    RunInContainer(ctx context.Context, namespace, podName, containerName string, cmd []string, attach api.AttachIO) error

    // 设置节点的参数，包含容量、condition等
    ConfigureNode(context.Context, *v1.Node)
}

// 返回Pod的统计
type PodMetricsProvider interface {
    GetStatsSummary(context.Context) (*statsv1alpha1.Summary, error)
}

// 用于支持Pod状态的异步更新
type PodNotifier interface {
    // 异步通知Pod的状态，注册回调函数，当Pod状态发生变化时就会调用回调函数
    NotifyPods(context.Context, func(*corev1.Pod))
}

type NodeProvider interface {
    // 用于探测节点是否存活，k8s周期调用该函数确定节点是否存活
    Ping(context.Context) error

    // 异步通知节点的状态，注册回调函数，当节点状态发生变化时就会调用回调函数
    NotifyNodeStatus(ctx context.Context, cb func(*corev1.Node))
}
```

以上接口中，除了Provider接口必须实现，其他接口都是可选的。

下面以CreatePod()为例看下azure的具体实现：

``` golang
func (p *ACIProvider) CreatePod(ctx context.Context, pod *v1.Pod) error {
    ...

    return p.createContainerGroup(ctx, pod.Namespace, pod.Name, &containerGroup)
}

func (p *ACIProvider) createContainerGroup(ctx context.Context, podNS, podName string, cg *aci.ContainerGroup) error {
    ctx = addAzureAttributes(ctx, span, p)

    cgName := containerGroupName(podNS, podName)
    _, err := p.aciClient.CreateContainerGroup(
        ctx,
        p.resourceGroup,
        cgName,
        *cg,
    )

    if err != nil {
        log.G(ctx).WithError(err).Errorf("failed to create container group %v", cgName)
    }

    return err
}
```

CreatePod()首先准备创建ACI容器组的资源，然后调用createContainerGroup()，该函数对接口调用再次封装，然后调用了aciClient的CreateContainerGroup()创建ACI容器组。而CreateContainerGroup()就是调用ACI的API接口创建容器组。PodLifecycleHandler中的函数实现方式都类似，只需要对接后端的接口即可。

Provider中剩下logs和exec则比较麻烦：

* GetContainerLogs 读取日志，如果需要支持`-f`选项就比较麻烦
* RunInContainer 执行命令，需要实时将命令的结果传送回来

上面两个函数都需要使用websocket实时回传数据，这个在这里就是另一个问题，这里不做过多介绍。

实现了Provider接口，剩下的只需实现PodNotifier中的NotifyPods()。

NotifyPods()用于异步通知Pod的状态变化。设想下k8s展示Pod状态的实现，k8s如何知道Pod的状态呢？一种方式是k8s定时调用GetPods()接口就得到当前节点的所有Pod，当Node和Pod较多时，资源消耗还是有些多的。另一种方式就是，节点通过比较k8s认为节点有的Pod和ACI上实际有的容器组，就得到应该更新哪些Pod的状态。

Pod的异步更新实现在provider/podsTracker.go：

``` golang
type PodsTrackerHandler interface {
    // 查询存活的Pod
    ListActivePods(ctx context.Context) ([]PodIdentifier, error)

    // 查询Pod的状态
    FetchPodStatus(ctx context.Context, ns, name string) (*v1.PodStatus, error)

    // 清理Pod
    CleanupPod(ctx context.Context, ns, name string) error
}

type PodsTracker struct {
    rm       *manager.ResourceManager
    updateCb func(*v1.Pod)
    handler  PodsTrackerHandler
}

// NotifyPods函数的实现，该函数只在VK节点的PodController启动时调用一次
func (p *ACIProvider) NotifyPods(ctx context.Context, notifierCb func(*v1.Pod)) {

    // Capture the notifier to be used for communicating updates to VK
    p.tracker = &PodsTracker{
        rm:       p.resourceManager,
        updateCb: notifierCb,
        handler:  p,
    }

    go p.tracker.StartTracking(ctx)
}
```

而在StartTracking()函数中会定时执行Pod的更新(updatePodsLoop)和删除(cleanupDanglingPods)：

``` golang
func (pt *PodsTracker) updatePodsLoop(ctx context.Context) {

    // 从资源管理器获取当前节点的Pod
    k8sPods := pt.rm.GetPods()
    for _, pod := range k8sPods {
        updatedPod := pod.DeepCopy()
        ok := pt.processPodUpdates(ctx, updatedPod)
        if ok {
            pt.updateCb(updatedPod)
        }
    }
}

// 处理Pod的更新，返回值表明Pod的状态是否更新
func (pt *PodsTracker) processPodUpdates(ctx context.Context, pod *v1.Pod) bool {

    // 调用ACI的接口获取Pod的状态
    podStatusFromProvider, err := pt.handler.FetchPodStatus(ctx, pod.Namespace, pod.Name)
    if err == nil && podStatusFromProvider != nil {
        // 如果获取状态没有出错，并且返回了状态，则将状态信息更新到Pod
        podStatusFromProvider.DeepCopyInto(&pod.Status)
        return true
    }

    if errdef.IsNotFound(err) || (err == nil && podStatusFromProvider == nil) {
        if pod.Status.Phase == v1.PodRunning {
            // 如果k8s中的状态是Running，但是ACI容器组不存在，则将k8s中容器的状态设置为Failed，此时会重建Pod
            pod.Status.Phase = v1.PodFailed
            pod.Status.Reason = statusReasonNotFound
            pod.Status.Message = statusMessageNotFound
            now := metav1.NewTime(time.Now())
            for i := range pod.Status.ContainerStatuses {
                if pod.Status.ContainerStatuses[i].State.Running == nil {
                    continue
                }

                // 更新Pod的状态
                pod.Status.ContainerStatuses[i].State.Terminated = &v1.ContainerStateTerminated{
                    ExitCode:    containerExitCodeNotFound,
                    Reason:      statusReasonNotFound,
                    Message:     statusMessageNotFound,
                    FinishedAt:  now,
                    StartedAt:   pod.Status.ContainerStatuses[i].State.Running.StartedAt,
                    ContainerID: pod.Status.ContainerStatuses[i].ContainerID,
                }
                pod.Status.ContainerStatuses[i].State.Running = nil
            }

            return true
        }

        return false
    }

    if err != nil {
        log.G(ctx).WithError(err).Errorf("failed to retrieve pod %v status from provider", pod.Name)
    }

    return false
}

// 删除Pod
func (pt *PodsTracker) cleanupDanglingPods(ctx context.Context) {

    // 获取k8s中的Pod和ACI容器组
    k8sPods := pt.rm.GetPods()
    activePods, err := pt.handler.ListActivePods(ctx)
    if err != nil {
        log.G(ctx).WithError(err).Errorf("failed to retrive active container groups list")
        return
    }

    if len(activePods) > 0 {
        for i := range activePods {
            // 遍历ACI容器组存活的Pod，如果k8s中没有对应的Pod，则删除ACI容器组
            pod := getPodFromList(k8sPods, activePods[i].namespace, activePods[i].name)
            if pod != nil {
                continue
            }

            err := pt.handler.CleanupPod(ctx, activePods[i].namespace, activePods[i].name)
            if err != nil && !errdef.IsNotFound(err) {
                log.G(ctx).WithError(err).Errorf("failed to cleanup pod %v", activePods[i].name)
            }
        }
    }
}
```

* updatePodsLoop：处理k8s中有，ACI容器组不存在的情况
* cleanupDanglingPods：处理ACI中有，k8s中不存在的情况

那么，virtual kubelet的整体结构如下：

![virtual kubelet](https://github.com/luofengmacheng/docker_doc/blob/master/kubernetes/pics/virtual_kubelet.png)

总结下上面的三条路径：

* PodController通过informer机制监听Pod的变化，然后执行Pod的增删改查操作
* virtual kubelet提供http接口，当用户执行`kubectl logs/exec`时，就调用对应的函数，然后会调用Provider接口中对应的函数，这里主要难点在于需要实时将数据回传，展示给用户
* PodNotifier提供了异步更新Pod的接口，apiserver为了让etcd中Pod的数据与节点上Pod的数据保持一致，会定时调用节点的接口查询Pod的状态，当节点和Pod比较多时，比较消耗apiserver的资源。为了节省资源，节点会比较k8s中的Pod的数据和后端实际Pod的数据，如果发现有不一致(k8s中有该Pod，后端没有；k8s中没有该Pod，后端有)，则执行状态的更新或者后端容器组的操作
