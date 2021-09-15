## 使用CloudProvider对接云厂商的LB

### 1 CloudProvider

CloudProvider，顾名思义，就是k8s提供给云的接口，让k8s能够与云对接。

k8s中LoadBalancer服务就是为了从外部访问k8s中的服务而设计的，它的工作方式如下：

当用户创建一个LoadBalancer类型的服务时，k8s会先创建一个ClusterIP的服务，此时也会生成一个ClusterIP。当k8s监听到服务类型为LoadBalancer时，会创建一个云厂商的LB，并且会分配一个LB的EIP，并将EIP填充到status.loadBalancer.ingress.ip，用户查看服务的EXTERNAL-IP列就可以查看到该EIP，后续可以通过该EIP访问服务。

在上述过程中，k8s需要创建云厂商的LB，但是不同云厂商的LB的实现有所区别，那么就需要有一种方式能够让k8s可以调用LB的接口进行操作。这就是CloudProvider的能力，或者说应该是Cloud Provider Interface，是将云厂商的LB对接到k8s时要实现的接口。

### 2 如何实现自己的CloudProvider



### 3 从CloudProvider到Cloud Controller Manager