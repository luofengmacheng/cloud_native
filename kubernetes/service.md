## kubernetes中的服务注册和服务发现

### 1 服务注册和服务发现

什么是服务？为什么会有服务，不能直接用IP和端口吗？

服务可以看做是一个标识，其它业务在访问该服务时可以在配置文件中配置该标识，而不用配置IP和端口。如果直接配置IP和端口，那么在对设备进行调整或者裁撤时，就需要修改其它业务的相关配置，在一个十分负载的系统中，这样的行为是不可接受的。

服务注册和服务发现：

其实可以理解为路由注册和路由发现。服务注册：在一个统一的服务管理中心进行注册，告知某个服务的标识和服务中的机器。服务发现：从服务管理中心获取一个服务的标识，将请求发送给该标识，由系统转发到实际处理请求的机器，或者通过标识获取服务的机器，然后再将请求发送给获取的机器。

### 2 服务注册(创建服务)

通过上面对服务的介绍，知道服务至少需要两个内容：服务名(服务标识号)和端口。

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: love-test
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx-rs-test
```

该服务的服务名是nginx-service，服务的端口是80，映射到后端实际pod的端口号也是80，通过`app=nginx-rs-test`选择后端的pod。

上面的服务创建过程有个问题：如果服务后端的pod的端口变更时，就需要调整服务中端口映射关系。为了解决这个问题，在设置端口时可以对端口命名：

* 在创建pod时需要设置容器的端口和对应的名字
* 服务的端口映射关系中，映射的目的端口是端口名字而不是具体的端口

### 3 服务发现

服务发现即通过某种方式访问服务，根据服务的访问方向，在k8s中有三种方式：

* 在`集群内部`连接`集群内部`的服务：ClusterIP
* 在`集群内部`连接`集群外部`的服务：ExternalName
* 在`集群外部`连接`集群内部`的服务：NodePort、LoadBalancer、Ingress

当然，就不会有在集群外部连接集群外部的服务了，不过这种方式也可以通过k8s做下代理。

#### 3.1 在`集群内部`连接`集群内部`的服务

1 通过ClusterIP

前面也介绍过服务的几种类型，默认创建的服务的类型是`ClusterIP`，也就是创建该服务后，k8s会给该服务分配一个内部的IP地址：ClusterIP，ClusterIP就相当于是该服务的标识号，之后就可以通过ClusterIP和服务的端口号进行访问了。

通过yaml文件创建服务后，可以通过`kubectl get svc`查看服务的ClusterIP。

2 通过环境变量

创建服务后，k8s会在pod的环境变量中加上服务的IP和端口：`NGINX_SERVICE_SERVICE_HOST`和`NGINX_SERVICE_SERVICE_PORT`。那么，就可以直接使用这两个变量作为服务的标识，本质上来说，这种方式也是通过ClusterIP进行访问的，只是通过一个固定标识去访问，因为ClusterIP也可能会改变。

3 通过FQDN(全限定域名)

k8s在启动容器后会修改容器的/etc/resolv.conf，其中主要有两个配置：

* nameserver dns域名服务器
* search 定义域名的搜索列表，default名字空间的容器的search会配置为`default.svc.cluster.local svc.cluster.local cluster.local`

FQDN其实就是域名而已，而在k8s中服务也可以通过域名访问，例如，上面创建的服务可以通过域名`nginx-service.love-test.svc.cluster.local`进行访问，可以登录到容器用curl进行测试。由于配置了search就可以直接用nginx-service作为域名访问。

search的作用是：当访问一个域名访问不到时就会尝试在search后面的域名下访问，例如，将nginx-service作为域名访问时，由于无法解析成IP地址，就会访问nginx-service.love-test.svc.cluser.local。

当然，用FQDN也有个问题：如果服务使用的不是默认端口，就需要使用其它方式获取服务的端口，例如，环境变量。

#### 3.2 在`集群内部`连接`集群外部`的服务

在集群内部访问集群外部的服务时，需要通过k8s将外部的实例映射成k8s内部的服务，这需要借助Endpoint对象完成。

Endpoint对象相当于是服务中的实例的列表，使用`kubectl describe service nginx-service`可以查看服务的Endpoint，Endpoint列表中就是每个实例的IP和端口号。当调用一个服务时，服务不是直接将请求转发给后端的机器，而是从Endpoint列表中获取一个实例进行转发。因此，只要将外部服务的实例填充到集群内部的服务的Endpoint，就可以通过服务进行转发。

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  ports:
  - port: 80
```

创建一个服务external-service，其中不配置selector，那么k8s就不会为该服务创建Endpoint。

``` yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: external-service
subsets:
  - addresses:
    - ip: 1.1.1.1
    - ip: 2.2.2.2
    ports:
    - port: 80
```

然后，我们创建一个Endpoint，其中主要的配置就是addresses，用于配置外部的实例的IP，那么Endpoint如何跟Service进行关联呢？其实就是通过名字关联的，也就是它们的metadata.name属性是一样的，这里是external-service。

除了使用Endpoint，对于外部域名的访问还可以在k8s集群中创建一个服务用于转发，这种方式的好处是将外部的域名访问隐藏在服务后面，之后服务的内容变更后，调用方不用修改，相当于加了一层代理：

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  type: ExternalName
  externalName: api.xxx.com
  ports:
  - port: 80
```

#### 3.3 在`集群外部`连接`集群内部`的服务

服务的类型除了上面的ClusterIP，还有另外的三种类型：

* NodePort 每个集群节点都相当于是服务的代理，每个集群节点都会打开一个端口，当访问某个节点的该端口时，k8s会将流量重定向到Endpoint中的其中一个实例。那么，就可以通过三种方式访问后端的实例：通过ClusterIP的方式；通过pod的IP和端口；通过集群的节点和专用端口。
* LoadBalancer 这种方式是NodePort的一种扩展，会在k8s集群前端再加一个云厂商提供的负载均衡器，负载均衡器会将收到的请求转发给后端的k8s的Node的NodePort端口
* Ingress：在集群中部署负载均衡器，并通过NodePort提供访问

1 NodePort

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: love-test
spec:
  type: NodePort
  ports:
  - port: 8080
    targetPort: 80
    nodePort: 31234
  selector:
    app: nginx-rs-test
```

跟之前ClusterIP的不同点在于service.spec.type和service.spec.ports.nodePort，将类型设置为NodePort，而nodePort设置为31234，当然，nodePort也可以不配置，那么k8s就会随机选择一个端口。该服务就可以通过以下方式访问：

```
1 ClusterIP
curl ClusterIP:8080
2 NodePort
curl kubenode:31234
```

上面的服务的yaml文件中有3个端口：

* port：ClusterIP的端口
* targetPort：对应到容器中应用的端口
* nodePort：node节点上启动的端口，由kube-proxy启动

2 Ingress

Ingress相比于前面的NodePort和LoadBalancer的主要区别在于：

* 虽然NodePort可以通过节点对服务进行负载均衡，但是还是需要在节点前面额外添加一层负载均衡器以应对节点的异常，实际中不会使用这种方式
* LoadBalancer需要云平台的支持，导致与云平台有一定的耦合，并且每个服务都需要负载均衡器以及公有IP

Ingress是k8s原生支持的负载均衡器，并且能够代理多种服务，是实际使用最多的提供服务的方式。

其实Ingress就是将常用的负载均衡软件(例如，nginx、haproxy)部署到k8s，这样做的好处是：

kube-proxy启动一个端口就可以处理很多服务的请求，因为请求的转发可以通过路由配置处理，相当于一个7层负载均衡器

### 4 服务的实现

#### 4.1 /etc/resolv.conf

当在Pod中访问服务名时，会将服务名当作域名进行解析，而域名解析过程中就会用到/etc/resolv.conf配置文件。

/etc/resolv.conf配置文件包含以下几个部分：

* nameserver DNS服务器，可以指定多个DNS服务器，当且仅当前一个域名服务器无响应时才访问下一个
* search domain1 domain2 domain3，当访问的域名无法解析时，会在域名后面加上domain1，再次尝试解析，依此进行尝试解析
* domain domain.com，默认域名，也就是当search没有设置时，search=domain，需要注意的是：search和domain不会同时配置

在Pod中通常都会配置nameserver和search：

* nameserver配置为DNS服务器的地址，也就是CoreDNS的地址
* search配置为服务域名，例如，`default.svc.cluster.local svc.cluster.local cluster.local`

因此，当访问service_name服务时，会尝试向CoreDNS发送解析service_name.default.svc.cluster.local域名的请求，当没有响应时，会尝试解析service_name.svc.cluster.local。

#### 4.2 CoreDNS

CoreDNS是一个使用golang开发的域名服务器，域名服务器的工作方式就是根据请求的域名返回对应的IP地址。

因此，CoreDNS只需要得到Service和ClusterIP的对应关系即可。

CoreDNS通过client-go的informer机制监视k8s中的Service和ClusterIP的变化，并保存这些数据用于域名解析，当收到域名解析的请求时，查询内部数据结构就可以得到对应的ClusterIP。

有了CoreDNS，并修改/etc/resolv.conf，就能够实现将ServiceName->ServiceIp的映射关系。当请求到达ServiceIP该怎么办呢？那就涉及到服务转发。

#### 4.3 服务转发

通过上面的介绍，实现服务的思路是：

* 维护服务VIP与后端pod的转发规则
* 将请求转发到后端pod

在k8s的实现过程中，服务的实现经历了3个阶段：

* userspace
* iptables
* ipvs

#### 4.3.1 用户空间

![userspace](https://github.com/luofengmacheng/cloud_native/blob/master/kubernetes/pics/service_userspace.png)

kube-proxy通过apiserver监听服务的状态变化，发现用户创建了redis服务，后端有2个pod，于是kube-proxy在本机起一个随机端口，然后在iptables中创建2条规则：

* redis_svc_ip:port -> kube-proxy:port 将redis的VIP转发到本机proxy的某个端口
* kube-proxy:port -> Pod-redis-1 & Pod-redis-2 将本机的某个端口转发到后端的redis的Pod

当集群中的其他Pod访问redis的VIP时，请求就会通过kube-proxy转发到后端的redis的Pod。

kube-proxy的作用是：负责监听服务和Pod的状态，维护iptables规则，并且服务的转发也通过了kube-proxy的端口

#### 4.3.2 iptables

![iptables](https://github.com/luofengmacheng/cloud_native/blob/master/kubernetes/pics/service_iptables.png)

kube-proxy通过apiserver监听服务的状态变化，发现用户创建了redis服务，后端有2个pod，于是kube-proxy在iptables中创建1条规则：

* redis_svc_ip:port -> Pod-redis-1 & Pod-redis-2

当集群中的其他Pod访问redis的VIP时，请求就会直接转发给后端的redis的Pod。

kube-proxy的作用是：负责监听服务和Pod的状态，维护iptables规则，但是服务的转发没有经过kube-proxy，而是直接在内核态进行转发，性能比userspace的方式更高。

#### 4.3.3 ipvs

ipvs的方式的流程比iptables一样，只是将iptables的规则替换为ipvs。

ipvs相比iptables的优势在于，当服务数量很多(例如>10000)时，iptables的性能会下降，而ipvs由于使用了内核哈希表，性能依然很高。

但是，ipvs在高性能场景下也会有其他问题：

* [深入kube-proxy ipvs模式的conn_reuse_mode问题](https://cloud.tencent.com/developer/article/1832908)

#### 4.4 NodePort

对于ClusterIP而言，通过上面的iptables或者ipvs就可以实现服务的转发。但是，对于NodePort、LoadBalancer、Ingress而言，情况有所不同。

NodePort相当于对ClusterIP的一种扩展，当创建NodePort服务时，会创建一个ClusterIP服务，然后在Node上面由kube-proxy开启外部服务端口，此时可以通过两种方式访问：

* 外部：Node:NodePort
* 内部：ServiceName、ClusterIp:ClusterPort

当kube-proxy收到请求后，通过访问的Node的Port就可以知道访问的服务，然后映射成ClusterIp:ClusterPort，再走原来的流程即可。

#### 4.5 LoadBalancer

#### 4.6 Ingress

### 5 探针(Probe)

前面已经介绍过存活探针，其实在k8s中还有另一种探针，用于表明容器已经准备好接收请求的就绪探针。存活探针和就绪探针在配置的使用过程基本一样：

* 都有HTTP、TCP、EXEC三种类型
* 都属于containers下面的一个字段：存活探针是livenessProbe，就绪探针是readinessProbe

只是它们的运作方式有些不一样：

* 存活探针探测失败表明服务已经不能正常工作；就绪探针探测失败表明该容器暂时还不能提供服务
* 如果存活探针探测失败，k8s会杀死容器并创建新的容器；如果就绪探针探测失败，k8s只是会将容器的IP从服务的Endpoint中移除