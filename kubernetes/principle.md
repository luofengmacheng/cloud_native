## Kubernetes工作原理

### 1 Master节点的组件

Master节点包含三个组件和存储组件(Master的组件不一定要跟存储组件部署在同一台机器)：

* apiserver：提供RESTful风格的操作的API，其它组件之间通过API进行交互
* scheduler：调度器，负责决策将Pod安排到哪个Node节点上运行
* controller-manager：控制器管理器，负责管理控制器，例如，RC，RS，Deployment等
* etcd：所有的资源对象和状态都会存储到etcd

### 2 Node节点的组件

Node节点包含自身的两个组件以及容器组件：

* kubelet：agent，负责管理容器的生命周期
* kube-proxy：代理，负责代理服务
* 容器运行时

### 3 请求的执行流程

kubectl是官方提供的管理集群的客户端，其实也是调用apiserver的API。那么，当使用kubectl创建Pod时，集群是如何完成该项工作的呢？

`在使用kubectl发起请求时，可以设置选项`--v=9`查看发起的请求的详细内容`

* 当使用kubectl创建Pod时，kubectl向apiserver发起创建Pod的请求，例如，发起在love-test的Namespace创建Pod的请求：`POST https://kubemaster.boss.com:6443/api/v1/namespaces/love-test/Pods 201 Created in 13 milliseconds`
在这行里面给出了发起请求的URL、方法、返回码和耗时，在POST的请求体中就是yaml文件的json格式。当apiserver返回201就认为Pod已创建。
* 当apiserver收到创建Pod的请求，其实就做了两件事：对请求进行验证(权限认证等)以及将Pod的对象写入etcd
* 当scheduler监听到(通过apiserver进行监听)etcd中没有关联Node节点的Pod，就会根据两阶段的调度策略选择合适的Node节点：预选(剔除掉不合适的Node节点)和优选(给Node节点打分，根据分数高低选择)。当scheduler确定了Pod要执行的Node后，会调用apiserver更新etcd中Pod的定义。
* 当kubelet监听到(通过apiserver进行监听)etcd中有Pod的Node节点是自身，并且Node节点上并没有运行该Pod，就会调用容器运行时运行容器。

以上的流程中需要明确以下几点：

* 不论是直接向apiserver发起请求，还是通过apiserver监听对象的状态变化，所有的组件的操作都只与apiserver交互
* 当通过apiserver监听对象状态变化时，每当更新对象(虽然定义对象时没有定义对象状态，状态也属于对象中的属性)时，apiserver会将所有的监听者发送更新后的对象

### 4 controller-manager(控制器管理器)

控制器管理器负责管理所有的控制器，而几乎所有的资源都有对应的控制器，例如，Replication Controller Controller(RC是一种资源，而RCC是管理RC的控制器)、Deployment Controller等，Replication Controller负责维护Pod的副本数，当Pod的实际副本数少于期望的副本数，RC就会创建新的Pod。

每个控制器都可以理解为一个协程，在每个协程中执行一个无限循环，通过订阅apiserver的变更通知，获知监听的对象的状态变化，然后跟期望的状态进行对比，最后做出一些操作，使得对象的状态向期望状态靠近。例如，RCC的工作方式：RCC会监听Pod对象的个数，获取Pod对象期望的个数，如果少于期望的个数，RCC会调用apiserver创建新的Pod，当然实际的Pod的调度和创建还是交给scheduler和kubelet。

### 5 kubelet

kubelet是安装在Node节点上的agent，主要职能为：

* 启动时在apiserver中注册Node对象资源
* 监听是否有Pod分配到当前节点，然后运行容器
* 上报容器的运行状态
* 当Pod被删除时，终止容器

总之，kubelet主要是负责Node对象监控和容器的生命周期管理。

kubelet可以从apiserver得到要运行的Pod的定义，然后运行容器，同时，它还可以运行本机的Pod的定义，并管理容器。

### 6 kube-proxy

kube-proxy的主要功能是负责服务的代理，当前的主要工作方式如下：

* 当kubectl创建一个服务后，Service控制器会分配一个虚拟IP，并创建Endpoint对象，而Endpoint控制器会根据Endpoint对象的配置填充其中的IP:Port
* kube-proxy会在Node节点上配置iptables，将虚拟IP的请求转发给Endpoint中的IP:Port
* 当某个容器访问某个服务时，数据包的目的地是虚拟的IP和端口，而在发送到网络之前，内核会根据iptables配置修改目的地为Endpoint中的IP:Port(选择的规则是随机的)

### 7 kubernetes集群的高可用

#### 7.1 客户应用的高可用

如果客户应用可以水平扩展(接入层和逻辑层)，可以使用Deployment保证应用的副本数以及应用的滚动更新。

如果客户应用不能水平扩展(存储层)，就需要采用选举机制，区分主从，其中的从要么只是等待主宕机，要么只能执行读操作。在kubernetes中，可以使用sidecar容器的机制进行选举，确认客户应用集群的主。

#### 7.2 集群自身的高可用

kubernetes集群自身的高可用跟上面客户应用的高可用比较类似：

* apiserver：无状态，直接运行多副本即可，不过前面要加上负载均衡
* scheduler和controller-manager：它们会根据监听的对象的状态做出相应的行为，如果运行多个实例会造成竞争，因此，这两个组件需要采用选举机制，多个实例时只有其中一个实例工作
* etcd：本身就是高可用的分布式键值存储系统，因此部署3+个奇数个节点即可

#### 7.3 scheduler和controller-manager的高可用

当启动多个scheduler和controller-manager时，系统已经做了选举，同一时刻只会有一个组件工作(其中的`--leader-elect`选项，默认为true)。而它们使用的选举机制是通过创建Endpoint对象实现的：

* 当启动组件时，会创建Endpoint对象：通过`kubectl get endpoints kube-scheduler -n kube-system -o yaml`命令查看，其中的`control-plane.alpha.kubernetes.io/leader`就是注册的信息
* `control-plane.alpha.kubernetes.io/leader`主要包含两个信息：holderIdentity(当前的主)，renewTime(最后更新时间)。成功创建Endpoint对象的组件会将自己的信息写入holderIdentity字段，并更新renewTime时间，之后会定期更新renewTime
* 如果主挂掉后，不会更新renewTime，当其它的从组件发现没有更新，就会尝试将自己的信息写入holderIdentity字段，并更新renewTime，从而会成为新的主

#### 7.4 高可用部署小结

下面对kubernetes集群的部署做个小结：

* 通常的高可用集群至少有3个节点(Master节点)
* 3个Master节点上部署kubelet，然后用crontab、systemd进行管理
* 3个Master节点上用kubelet读取本地的apiserver、scheduler、controller-manager、etcd的Pod的文件创建三个Master节点，所有的Master组件以容器运行
* Node节点同样需要部署kubelet，另外还需要部署kube-proxy，而kube-proxy同样以容器的方式运行

总之，所有的组件中，只有kubelet以宿主机进程运行，其它的组件都以容器运行