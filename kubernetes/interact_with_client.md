## 使用client-go与kubernetes集群交互

### 1 client-go简介及使用流程

client-go是kubernetes官方提供的kubernetes的api库，通过它可以调用kubernetes的api。

使用client-go调用kubernetes分成三个部分：

* 根据kubeconfig创建客户端
* 生成对应的资源对象
* 通过客户端调用接口操作资源对象

调用kubernetes.NewForConfig(c *rest.Config)根据配置对象创建客户端，为了生成rest.Config，有两种方式：一种是调用clientcmd.BuildConfigFromFlags("", *kubeconfig)直接从配置文件读取；另一种是调用clientcmd.RESTConfigFromKubeConfig()从参数的字符串读取。

client-go对资源的操作接口分成两种形式：一种是静态的，操作时需要提供明确的结构体数据；另一种是动态的，操作时只需要提供一个`map[string]interface{}`的对象。在使用kubectl进行资源的创建、更新、删除操作时，只需要提供yaml文件即可，k8s自己会去解析字段的值，因此，在进行资源的CUD时，客户端只需要提供yaml文件，而资源的查询操作，由于需要获取某个字段的值，而且不同的资源的结构也是不同的，使用动态接口就不太方便。

有了客户端，下面就需要生成对应的资源对象，有两种形式的资源操作接口，如果是用户提交的方式，肯定是用动态的，如果是用程序创建，肯定是用动态的。

动态接口：

``` golang
dynamicClient, err := dynamic.NewForConfig(restConfig)
if err != nil {
    // TODO: handle error
}

// data是从yaml文件读取的数据
// unstructured.Unstructured实际上就是个map[string]interface{}
// 在读取时，考虑到有多个资源对象的情况，因此，使用for循环进行了读取
var resources []unstructured.Unstructured
d := yaml.NewYAMLReader(bufio.NewReader(bytes.NewReader(data)))
for {
	data, err := d.Read()
	if err != nil {
		if err != io.EOF {
			return nil, errors.Wrap(err, "decode resource yaml")
		}
		break
	}
	var item unstructured.Unstructured
	if err := sigsyaml.Unmarshal(data, &item); err != nil {
		return nil, errors.Wrap(err, "unmarshal unstructured")
	}

    kind := item.GetKind()
    namespace := item.GetNamespace()

    // 根据kind获取对应的Group和Version
    group := getGroup(kind)
    version := getVersion(kind)
    gvr := schema.GroupVersionResource{Group: group, Version: version, Resource: kind}

    // 为了减少出现问题的概率，最好的办法是先读取完整个yaml文件，然后再调用接口，
    // 否则，如果文件中部分数据不合法，有可能造成部分资源创建成功，部分资源创建失败
    _, err := dynamicClient.Resource(gvr).Namespace(namespace).Create(ctx, &item, metav1.CreateOptions{})
    if err != nil {
        // TODO: handle error
    }
}
```

静态接口：

``` golang
type Deployment struct {
	Kind              string
	Namespace         string
	Name              string
	CreationTimestamp string
	Replica           int32
	AvailableReplicas int32
	Images            []string
}

k8sClient, err := kubernetes.NewForConfig(restConfig)
if err != nil {
    // TODO: handle error
}

itemList, err := k8sClient.AppsV1().Deployments(namespace).List(ctx, metav1.ListOptions{})
if err != nil {
	// TODO: handle error
}

// items中保存到就是Deployment的数组，可以返回给用户查看
items := []Deployment{}
for _, v := range itemList.Items {
	item := Deployment{
		Kind:              "Deployment",
		Namespace:         v.Namespace,
		Name:              v.Name,
		CreationTimestamp: v.CreationTimestamp.Format("2006-01-02 15:04:05"),
		Replica:           v.Status.Replicas,
		AvailableReplicas: v.Status.AvailableReplicas,
    }
	for _, c := range v.Spec.Template.Spec.Containers {
		item.Images = append(item.Images, c.Image)
	}
	items = append(items, item)
}
```

### 2 使用client-go实现控制器

client-go除了调用kubernetes的接口用于资源操作，另一个很重要的场景是实现控制器。

在kubernetes中，控制器是主要逻辑的实现部分，它的工作方式是：

* 监听资源的变化状态，并获取资源的当前状态
* 获取资源的期望状态
* 执行操作让当前状态向期望状态转移

再落实到具体到实现，client-go中也提供了相应的组件：

* Informer：负责跟踪资源的状态变化
* Indexer：在本地存储资源的最新状态
* Workqueue：

因此，使用client-go实现controller的具体逻辑就是：

* Controller通过Informer监听资源状态变化，将资源存储到本地的store中(通过Indexer操作store)
* 如果Informer监听到资源状态发生变化，会调用事先注册的ResourceEventHandler回调函数
* ResourceEventHandler回调函数将关心变更的Object放到Workqueue中
* Controller从Workqueue取出Object，启动worker执行业务逻辑使得当前状态向期望状态转移
* 在worker中使用lister获取resource，不需要频繁访问apiserver

[Writing Controllers](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-api-machinery/controllers.md)

[workqueue](https://github.com/kubernetes/client-go/blob/b350fc31ceb9df100385e2900b5987689df38e48/examples/workqueue/main.go)

``` golang
// Controller demonstrates how to implement a controller with client-go.
type Controller struct {
	indexer  cache.Indexer // 
	queue    workqueue.RateLimitingInterface // 工作队列
	informer cache.Controller // 监听资源变化
}

// 创建控制器
func NewController(queue workqueue.RateLimitingInterface, indexer cache.Indexer, informer cache.Controller) *Controller {
	return &Controller{
		informer: informer,
		indexer:  indexer,
		queue:    queue,
	}
}

func (c *Controller) processNextItem() bool {
	// 从队列中读取元素
	key, quit := c.queue.Get()
	if quit {
		return false
	}
	// Tell the queue that we are done with processing this key. This unblocks the key for other workers
	// This allows safe parallel processing because two pods with the same key are never processed in
	// parallel.
	// 
	defer c.queue.Done(key)

	// Invoke the method containing the business logic
	err := c.syncToStdout(key.(string))
	// Handle the error if something went wrong during the execution of the business logic
	c.handleErr(err, key)
	return true
}

// 控制器的具体业务逻辑
// 1 根据key使用Indexer从store中获取对应的对象
// 2 如果不存在，则打印相应的信息；如果存在，则打印一条提示信息
// 注意：这里不要执行重试逻辑
func (c *Controller) syncToStdout(key string) error {
	obj, exists, err := c.indexer.GetByKey(key)
	if err != nil {
		klog.Errorf("Fetching object with key %s from store failed with %v", key, err)
		return err
	}

	if !exists {
		fmt.Printf("Pod %s does not exist anymore\n", key)
	} else {
		// 这里需要根据UID判断资源是否是重新创建的
		fmt.Printf("Sync/Add/Update for Pod %s\n", obj.(*v1.Pod).GetName())
	}
	return nil
}

// 如果出错了需要执行重试逻辑
func (c *Controller) handleErr(err error, key interface{}) {
	if err == nil {
		// Forget about the #AddRateLimited history of the key on every successful synchronization.
		// This ensures that future processing of updates for this key is not delayed because of
		// an outdated error history.
		c.queue.Forget(key)
		return
	}

	// This controller retries 5 times if something goes wrong. After that, it stops trying.
	if c.queue.NumRequeues(key) < 5 {
		klog.Infof("Error syncing pod %v: %v", key, err)

		// Re-enqueue the key rate limited. Based on the rate limiter on the
		// queue and the re-enqueue history, the key will be processed later again.
		c.queue.AddRateLimited(key)
		return
	}

	c.queue.Forget(key)
	// Report to an external entity that, even after several retries, we could not successfully process this key
	runtime.HandleError(err)
	klog.Infof("Dropping pod %q out of the queue: %v", key, err)
}

// Run begins watching and syncing.
func (c *Controller) Run(workers int, stopCh chan struct{}) {
	defer runtime.HandleCrash()

	// Let the workers stop when we are done
	defer c.queue.ShutDown()
	klog.Info("Starting Pod controller")

	go c.informer.Run(stopCh)

	// Wait for all involved caches to be synced, before processing items from the queue is started
	if !cache.WaitForCacheSync(stopCh, c.informer.HasSynced) {
		runtime.HandleError(fmt.Errorf("Timed out waiting for caches to sync"))
		return
	}

	for i := 0; i < workers; i++ {
		go wait.Until(c.runWorker, time.Second, stopCh)
	}

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

	// 根据配置文件生成配置对象
	config, err := clientcmd.BuildConfigFromFlags(master, kubeconfig)
	if err != nil {
		klog.Fatal(err)
	}

	// 创建clientset
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		klog.Fatal(err)
	}

	// 创建POD的监听器
	podListWatcher := cache.NewListWatchFromClient(clientset.CoreV1().RESTClient(), "pods", v1.NamespaceDefault, fields.Everything())

	// 创建工作队列
	queue := workqueue.NewRateLimitingQueue(workqueue.DefaultControllerRateLimiter())

	// Bind the workqueue to a cache with the help of an informer. This way we make sure that
	// whenever the cache is updated, the pod key is added to the workqueue.
	// Note that when we finally process the item from the workqueue, we might see a newer version
	// of the Pod than the version which was responsible for triggering the update.

	unc(lw cache.ListerWatcher, objType runtime.Object, resyncPeriod time.Duration, h cache.ResourceEventHandler, indexers cache.Indexers) (cache.Indexer, cache.Controller)

NewIndexerInformer returns a Indexer and a controller for populating the index
while also providing event notifications. You should only used the returned
Index for Get/List operations; Add/Modify/Deletes will cause the event
notifications to be faulty.

Parameters:
 * lw is list and watch functions for the source of the resource you want to
   be informed of.
 * objType is an object of the type that you expect to receive.
 * resyncPeriod: if non-zero, will re-list this often (you will get OnUpdate
   calls, even if nothing changed). Otherwise, re-list will be delayed as
   long as possible (until the upstream source closes the watch or times out,
   or you stop the controller).
 * h is the object you want notifications sent to.
 * indexers is the indexer for the received object type.
    // 
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
			// IndexerInformer uses a delta queue, therefore for deletes we have to use this
			// key function.
			key, err := cache.DeletionHandlingMetaNamespaceKeyFunc(obj)
			if err == nil {
				queue.Add(key)
			}
		},
	}, cache.Indexers{})

	controller := NewController(queue, indexer, informer)

	// We can now warm up the cache for initial synchronization.
	// Let's suppose that we knew about a pod "mypod" on our last run, therefore add it to the cache.
	// If this pod is not there anymore, the controller will be notified about the removal after the
	// cache has synchronized.
	indexer.Add(&v1.Pod{
		ObjectMeta: meta_v1.ObjectMeta{
			Name:      "mypod",
			Namespace: v1.NamespaceDefault,
		},
	})

	// 启动控制器
	stop := make(chan struct{})
	defer close(stop)
	go controller.Run(1, stop)

	select {}
}
```

### 3 参考文档

* [client-go Examples](https://github.com/kubernetes/client-go/tree/b350fc31ceb9df100385e2900b5987689df38e48/examples)
* [k8s-client-go demo](https://github.com/owenliang/k8s-client-go)
