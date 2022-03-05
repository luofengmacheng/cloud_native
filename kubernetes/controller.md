## kubernetes中的Controller

### 1 什么是Controller？

kubernetes采用了声明式API，与声明式API相对应的是命令式API：

* 声明式API：用户只需要告诉期望达到的`结果`，系统自动去完成用户的期望
* 命令式API：用户需要关注`过程`，通过命令一步一步完成用户的需求

因此，用户向k8s提交的yaml文件中最重要的部分就是spec，相当于就是用户期望的结果，而使用`-o yaml`选项查看时，还有一个很重要的部分就是status，它表示的就是当前状态，因此，k8s主要任务就是完成`status->spec`的转变。这项工作就是Controller(控制器)完成的。

对于不同的资源，控制逻辑是不一样的，因此，就有很多Controller，例如，DeploymentController负责将Deployment的status向spec进行转变，ReplicaSetController负责将ReplicaSet的status向spec进行转变。

从上面可以看出，Controller的工作方式如下：

* 监听资源变化
* 得到资源的当前状态status和期望状态spec
* 执行逻辑使得status->spec

下面以Deployment的创建操作为例说明整个流程：

![创建Deployment的整体流程](https://github.com/luofengmacheng/docker_doc/blob/master/kubernetes/pics/controller1.png)

通过上图重新复习下各组件的工作方式：

* apiserver：为其他组件提供接口，并且所有的组件都通过apiserver进行交互
* etcd：存储集群的资源对象
* Controller Manager：管理控制器，Watch -> Analyze -> Act，监听资源的变化，分析出spec和status的差别，执行操作使得status向spec转变
* Scheduler：监听资源的变化，如果发现未调度的Pod，通过一定的策略选择出Node，设置Pod的Node字段
* Kubelet：监听调度给当前Node的Pod，并执行对应的操作

可以发现，除了apiserver和etcd，其他组件都可以称为Controller。

### 2 Controller的实现

知道了Controller的工作方式，如果是我们自己实现Controller，可以会这样来实现：

![自己实现Controller](https://github.com/luofengmacheng/docker_doc/blob/master/kubernetes/pics/controller2.png)

Controller直接通过Apiserver的接口监控对应资源的变化，当资源发生变化时，直接执行对应的业务逻辑，也就是调协循环。

这样会有啥问题呢？

当集群中Node很多时，就会有很多kubelet监控Pod的状态变化，而所有的监听操作都需要通过apiserver，那么apiserver的压力就会很大，就会造成集群的不稳定。

当然，其他资源(例如，Pod或者服务)很多时，同样会造成集群不稳定。

因此，k8s的client-go([client-go](https://github.com/kubernetes/client-go))库采用了另外的设计：

![Controller by k8s](https://github.com/kubernetes/sample-controller/blob/master/docs/images/client-go-controller-interaction.jpeg)

client-go components：

* Reflector：对特定类型的资源执行ListAndWatch，当监听到资源变更时，通过API获取最新的资源对象，然后将它放到Delta Fifo queue队列中
* Informer：从Delta Fifo queue队列中弹出对象，然后调用Indexer放到Store里面，同时调用用户提交的回调函数(ResourceEventHandler)
* Indexer：用于操作Store中的对象

Custom Controller components：

* Informer Reference和Indexer Reference都是对client-go中对象的引用，用户控制器可以通过cache库直接创建或者使用Factory工厂函数创建
* ResourceEventHandler：用户控制器接收对象的回调函数，一般来说，里面的逻辑就是，获取对象的key，然后将key写入WorkQueue
* WorkQueue：用户控制器创建的队列，负责存储用户控制器需要处理的对象的key
* Process Item：从WorkQueue中读取key，通过key获取对应的对象

上图是通常会给出的关于Controller的实际实现的逻辑，初看还是挺复杂的，大致的模块和功能如下：

![Controller by lf](https://github.com/luofengmacheng/docker_doc/blob/master/kubernetes/pics/controller3.png)

于是，Controller实现的步骤如下：

* 获取Informer和Indexer的引用，指定要监控变更的资源类型，注册ResourceEventHandler，并创建WorkQueue，用上述的3个对象初始化我们自己的Controller
* 编写Process Item Loop，从WorkQueue中读取key，然后执行我们自己的业务逻辑

因此，整个Controller我们需要注入的逻辑只有2个部分，其他都是相对固定的：

* ResourceEventHandler
* Process Item

### 3 Controller的使用

上面介绍了k8s中的Controller的实现，而要使用

下面对client-go中的workqueue的例子进行分析：

[workqueue example by client-go](https://github.com/kubernetes/client-go/blob/b350fc31ceb9df100385e2900b5987689df38e48/examples/workqueue/main.go)

``` golang
type Controller struct {
	indexer  cache.Indexer // Indexer，缓存的索引
	queue    workqueue.RateLimitingInterface // 带限速功能的WorkQueue
	informer cache.Controller // Informer
}

// 创建控制器
func NewController(queue workqueue.RateLimitingInterface, indexer cache.Indexer, informer cache.Controller) *Controller {
	return &Controller{
		informer: informer,
		indexer:  indexer,
		queue:    queue,
	}
}

// worker的具体执行逻辑
func (c *Controller) processNextItem() bool {
	// 从workqueue中获取key
	key, quit := c.queue.Get()
	if quit {
		return false
	}
	
	// 告诉队列已经处理完毕
	defer c.queue.Done(key)

	err := c.syncToStdout(key.(string))
	
	// 错误处理
	c.handleErr(err, key)
	return true
}

// 控制器的业务逻辑，这里就执行status->spec的转变
func (c *Controller) syncToStdout(key string) error {
	obj, exists, err := c.indexer.GetByKey(key)
	if err != nil {
		klog.Errorf("Fetching object with key %s from store failed with %v", key, err)
		return err
	}

	if !exists {
		// Pod已经不存在
		fmt.Printf("Pod %s does not exist anymore\n", key)
	} else {
		// 这里执行status->spec的转变逻辑
		fmt.Printf("Sync/Add/Update for Pod %s\n", obj.(*v1.Pod).GetName())
	}
	return nil
}

// 错误处理，包含重试处理
func (c *Controller) handleErr(err error, key interface{}) {
	if err == nil {
		// 处理完毕
		c.queue.Forget(key)
		return
	}

	// 如果出现问题，会进行重试，也就是重新入workqueue
	// 但是，入workqueue不超过5次
	if c.queue.NumRequeues(key) < 5 {
		klog.Infof("Error syncing pod %v: %v", key, err)

		// 重新入workqueue
		c.queue.AddRateLimited(key)
		return
	}

	c.queue.Forget(key)
	runtime.HandleError(err)
	klog.Infof("Dropping pod %q out of the queue: %v", key, err)
}

// 启动我们自己的控制器
func (c *Controller) Run(workers int, stopCh chan struct{}) {
	defer runtime.HandleCrash()

	defer c.queue.ShutDown()

	// 启动Informer开始监听资源变化
	go c.informer.Run(stopCh)

	// 等待cache同步
	if !cache.WaitForCacheSync(stopCh, c.informer.HasSynced) {
		runtime.HandleError(fmt.Errorf("Timed out waiting for caches to sync"))
		return
	}

	// 运行若干个worker，
	// wait.Until()，每隔1秒执行runWorker()函数，直到stopCh收到结束信号
	for i := 0; i < workers; i++ {
		go wait.Until(c.runWorker, time.Second, stopCh)
	}

	// 读取结束信号，结束控制器
	<-stopCh
	klog.Info("Stopping Pod controller")
}

func (c *Controller) runWorker() {
	for c.processNextItem() {
	}
}

func main() {
	var kubeconfig string
	var master string

	flag.StringVar(&kubeconfig, "kubeconfig", "", "absolute path to the kubeconfig file")
	flag.StringVar(&master, "master", "", "master url")
	flag.Parse()

	// 通过master和kubeconfig生成配置对象
	config, err := clientcmd.BuildConfigFromFlags(master, kubeconfig)
	if err != nil {
		klog.Fatal(err)
	}

	// 根据配置对象生成clientset，用于连接k8s
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		klog.Fatal(err)
	}

	// 创建Pod的watcher
	podListWatcher := cache.NewListWatchFromClient(clientset.CoreV1().RESTClient(), "pods", v1.NamespaceDefault, fields.Everything())

	// 创建workqueue
	queue := workqueue.NewRateLimitingQueue(workqueue.DefaultControllerRateLimiter())

	// 创建Indexer和Informer，其中重要的是两个参数，Pod的watcher和回调函数
	// 告知Informer，我们只监听Pod的资源变化，并且，给Infomer注册回调函数
	indexer, informer := cache.NewIndexerInformer(podListWatcher, &v1.Pod{}, 0, cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			key, err := cache.MetaNamespaceKeyFunc(obj)
			if err == nil {
				queue.Add(key)
			}
		},
		UpdateFunc: func(old interface{}, new interface{}) {
			key, err := cache.MetaNamespaceKeyFunc(new)
			if err == nil {
				queue.Add(key)
			}
		},
		DeleteFunc: func(obj interface{}) {
			key, err := cache.DeletionHandlingMetaNamespaceKeyFunc(obj)
			if err == nil {
				queue.Add(key)
			}
		},
	}, cache.Indexers{})

	// 创建我们自己的控制器
	controller := NewController(queue, indexer, informer)

	// 启动控制器
	stop := make(chan struct{})
	defer close(stop)
	go controller.Run(1, stop)

	// Wait forever
	select {}
}
```

### 参考资料：

1 [client-go under the hood](https://github.com/kubernetes/sample-controller/blob/master/docs/controller-client-go.md)

2 [client-go Examples](https://github.com/kubernetes/client-go/tree/master/examples)

3 [k8s-client-go demo](https://github.com/owenliang/k8s-client-go)

4 [writing controllers](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-api-machinery/controllers.md)