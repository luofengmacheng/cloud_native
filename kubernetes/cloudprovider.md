## 使用CloudProvider对接云厂商的LB

### 1 CloudProvider

CloudProvider，顾名思义，就是k8s提供给云的接口，让k8s能够与云对接。

k8s中LoadBalancer服务就是为了从外部访问k8s中的服务而设计的，它的工作方式如下：

当用户创建一个LoadBalancer类型的服务时，k8s会先创建一个ClusterIP的服务，此时也会生成一个ClusterIP。当k8s监听到服务类型为LoadBalancer时，会创建一个云厂商的LB，并且会分配一个LB的EIP，并将EIP填充到status.loadBalancer.ingress.ip，用户查看服务的EXTERNAL-IP列就可以查看到该EIP，后续可以通过该EIP访问服务。

在上述过程中，k8s需要创建云厂商的LB，但是不同云厂商的LB的实现有所区别，那么就需要有一种方式能够让k8s可以调用LB的接口进行操作。这就是CloudProvider的能力，或者说应该是Cloud Provider Interface，是将云厂商的LB对接到k8s时要实现的接口。

### 2 如何实现自己的CloudProvider

由于当前CloudProvider主要用于公有云LB的对接，因此，这里的实现仍然以LB对接为主。

CPI提供了cloudprovider.Interface：

``` golang
// k8s.io/cloud-provider/cloud.go
type Interface interface {
	// 初始化操作，一般用于构建客户端连接
	Initialize(clientBuilder ControllerClientBuilder, stop <-chan struct{})
	// 返回LoadBalancer接口，这里是
	LoadBalancer() (LoadBalancer, bool)
	// Instances returns an instances interface. Also returns true if the interface is supported, false otherwise.
	Instances() (Instances, bool)
	// Zones returns a zones interface. Also returns true if the interface is supported, false otherwise.
	Zones() (Zones, bool)
	// Clusters returns a clusters interface.  Also returns true if the interface is supported, false otherwise.
	Clusters() (Clusters, bool)
	// Routes returns a routes interface along with whether the interface is supported.
	Routes() (Routes, bool)
	// cloudprovider的名字
	ProviderName() string
	// HasClusterID returns true if a ClusterID is required and set
	HasClusterID() bool
}
```

其中最重要的就是LoadBalancer()函数，该函数返回一个LoadBalancer接口：

``` golang
// k8s.io/cloud-provider/cloud.go
type LoadBalancer interface {
	// 查询服务对应的LB，如果存在，则返回它的状态(LB VIP)
	GetLoadBalancer(ctx context.Context, clusterName string, service *v1.Service) (status *v1.LoadBalancerStatus, exists bool, err error)
	// 查询服务对应的LB的名字
	GetLoadBalancerName(ctx context.Context, clusterName string, service *v1.Service) string
	// 创建服务对应的LB，并返回它的状态，创建LoadBalancer类型的服务时调用
	EnsureLoadBalancer(ctx context.Context, clusterName string, service *v1.Service, nodes []*v1.Node) (*v1.LoadBalancerStatus, error)
	// 更新服务对应的LB的后端的RS，更新服务时调用
	UpdateLoadBalancer(ctx context.Context, clusterName string, service *v1.Service, nodes []*v1.Node) error
	// 删除服务对应的LB，删除服务时调用
	EnsureLoadBalancerDeleted(ctx context.Context, clusterName string, service *v1.Service) error
}
```

因此，为了对接公有云厂商的LB，需要做两件事：实现LoadBalancer接口；启用cloudprovider.Interface。

``` golang
type MyProvider struct {
	vpcId     string
	subnetId  string
	// 用户可以在此处添加一些相关属性
}

func NewProvider() cloudprovider.Interface {
	p := &MyProvider{
		vpcId:     os.Getenv("VPC_ID"),
		subnetId:  os.Getenv("SUBNET_ID"),
	}

	return p
}

// 基本cloudprovider.Interface的所有方法都直接返回，因为MyProvider实现了所有方法
func (p *MyProvider) LoadBalancer() (cloudprovider.LoadBalancer, bool) {
	return p, true
}

func (p *MyProvider) GetLoadBalancer(ctx context.Context, clusterName string, svc *v1.Service) (status *v1.LoadBalancerStatus, exists bool, err error) {
	// 获取服务对应的LB(为了实现该功能，需要将服务与LB绑定，可以在服务的注解中保存LB的信息，可以是LB的资源ID)
}

func (p *MyProvider) EnsureLoadBalancer(ctx context.Context, clusterName string, svc *v1.Service, nodes []*v1.Node) (*v1.LoadBalancerStatus, error) {
	// 1 从服务的注解中提取出LB的配置
	// 2 查询服务是否有对应的LB，如果没有则创建LB
	// 3 获取EIP
	// 4 给LB绑定EIP、后端VS、后端RS
	// 5 返回EIP
}

func (p *MyProvider) UpdateLoadBalancer(ctx context.Context, clusterName string, svc *v1.Service, nodes []*v1.Node) error {
	// 跟创建操作类似，需要更新LB后端的VS和RS
}

func (p *MyProvider) EnsureLoadBalancerDeleted(ctx context.Context, clusterName string, svc *v1.Service) error {
	// 删除LB，在删除LB前需要确保其他额外的资源已经被删除
}
```

上面的代码实现了cloudprovider.Interface和LoadBalancer，其中LoadBalancer对接了公有云LB的接口。

k8s中最常用的就是控制器模式：监听资源变化，当资源变化时执行对应的操作。上述逻辑其实也可以通过控制器模式插入到k8s中。

k8s中service控制器的创建函数：

``` golang
// k8s.io/kubernetes/pkg/controller/service
func New(
	cloud cloudprovider.Interface,
	kubeClient clientset.Interface,
	serviceInformer coreinformers.ServiceInformer,
	nodeInformer coreinformers.NodeInformer,
	clusterName string,
) (*Controller, error) {
}
```

从函数的入参可以看出来，cloud传递的就是cloudprovider.Interface。因此，为了能够调用我们实现的MyProvider，只要将对象传给service.New()就行。

``` golang
import "k8s.io/client-go/informers"
import "k8s.io/kubernetes/pkg/controller/service"

sharedInformers := informers.NewSharedInformerFactory(k8s, ResyncPeriod)

p := NewProvider()

svc, err := service.New(
	p,
	k8sClient, // k8s的客户端
	sharedInformers.Core().V1().Services(),
	sharedInformers.Core().V1().Nodes(),
"")

stopCh := make(chan struct{})
go svc.Run(done, 5)

select {
	case <-done:
		break
}
```

上述代码先创建informer和cloudprovider，然后创建service，启动service即可。在service的执行过程中，会在适当的时机去调用LoadBalancer接口中的方法。

将上述程序编译后，直接部署到k8s中使用即可。

### 3 从CloudProvider到Cloud Controller Manager

CloudProvider的功能本来是提供给云厂商的LB接入的，但是由于跟k8s的核心代码耦合比较紧。为了减小跟核心代码的耦合，对CloudProvider进行重构，并且独立出来，独立出来的部分就称为Cloud Controller Manager(CCM)。

其实新的名字可能跟贴近该模块的功能：云控制器管理器，用于管理云厂商的控制器。