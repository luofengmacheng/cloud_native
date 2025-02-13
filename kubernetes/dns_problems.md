## 集群中的DNS配置以及问题定位

### 1 背景

集群环境中，当在Pod中访问服务时，会通过CoreDNS提供的DNS服务将服务名转换成IP地址，然后Pod再访问IP地址完成请求。

本文我们会学习集群中的DNS工作原理以及问题排查方法。

### 2 CoreDNS的关键配置说明

CoreDNS包含以下资源：

* ConfigMap：coredns
* Deployment：coredns
* Service：kube-dns

ConfigMap中指包含一个配置文件Corefile：

```
Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
```

Corefile配置文件中的字段含义：

* :53表示监听53端口
* errors表示输出错误信息
* health表示检查服务状态
* ready表示启用就绪探针
* kubernetes行表示解析cluster.local域的DNS查询，后面的in-addr.arpa和ip6.arpa表示会进行反向查询(根据IP地址查找域名)
* pods insecure表示允许解析Pod IP
* fallthrough表示当查询不匹配时，转发给谁解析
* prometheus行表示通过9153端口向prometheus暴露指标
* forward行表示当在集群中查询失败时将请求发送给/etc/resolv.conf
* cache 30表示对条目进行缓存
* loop表示防止无限循环解析
* reload表示支持配置热加载
* loadbalance表示支持DNS负载均衡

CoreDNS的Deployment就是部署coredns的Pod，这里只需要注意两个地方：

* replicas设置为2，也就是会启动两个Pod，这两个Pod进行负载均衡，提供DNS服务
* dnsPolicy设置为Default

### 3 Pod的DNS策略

`pod.spec.dnsPolicy`用于设置Pod的DNS策略，它有4个值：

* Default：Pod从所在的节点继承解析配置，例如，/etc/hosts和/etc/resolv.conf
* ClusterFirst(默认的DNS策略)：使用集群的DNS配置
* ClusterFirstWithHostNet：对于以hostNetwork方式运行的Pod，应将其DNS策略显示设置为ClusterFirstWithHostNet，否则，以hostNetwork方式和ClusterFirst策略运行的Pod的DNS策略与Default策略一致
* None：此设置允许Pod忽略k8s环境中的DNS设置，Pod会使用其dnsConfig字段提供的DNS设置

dnsPolicy字段比较常见的是两个值：

* Default，使用宿主机的DNS解析配置，CoreDNS的Pod就是使用这种策略
* ClusterFirst，使用集群的DNS配置，普通的Pod通常用这种策略，这也是默认的DNS策略

### 4 一次完整的DNS域名查询

当程序需要解析域名时，会通过两个文件进行解析：/etc/hosts和/etc/resolv.conf，其中，/etc/hosts记录的是域名和IP的直接映射关系，/etc/resolv.conf记录的是域名服务器和根域名，会先尝试使用/etc/hosts文件进行解析，解释失败再使用/etc/resolv.conf中的域名服务器查询。

在集群中创建Pod时，如果未指定dnsPolicy字段，就会使用设置为默认值ClusterFirst，此时，/etc/resolv.conf中的域名服务器就是kube-dns服务的IP地址，而kube-dns后端的Pod就是CoreDNS的Pod。

当程序需要解析域名时，默认容器中的/etc/hosts中只有localhost和pod名称与IP的映射关系，因此，会查询/etc/resolv.conf中指定的域名服务器，该域名服务器IP就是kube-dns服务的IP地址，后端就是CoreDNS的Pod，因此，CoreDNS的Pod会收到域名查询请求。CoreDNS查询本地的数据，如果查询到对应的域名则返回，如果查询不到，则查询宿主机的/etc/resolv.conf。这就解释了为什么在Pod中既可以访问集群中的域名，也能够访问宿主机环境的域名。

### 5 关于dnsConfig

dnsPolicy使得我们能够取设置Pod是使用集群的域名服务器还是宿主机环境的域名服务器，但是有时候可能需要自定义Pod的域名服务器，这可以通过dnsConfig实现。

`pod.spec.dnsConfig`可以配置/etc/resolv.conf中的三个参数：nameservers、options、searches。

使用该参数时要注意两个地方：

* 如果dnsPolicy设置为None时，必须配置dnsConfig
* 如果dnsPolicy设置`非None`时，k8s会将dnsConfig中的配置追加到容器的/etc/resolv.conf，会删除相同的条目

### 6 测试Pod

当集群中出现DNS解析相关问题时，可以先创建以下的Pod，该Pod有相关的诊断工具。

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: dnsutils
  namespace: default
spec:
  containers:
  - name: dnsutils
    image: e2eteam/jessie-dnsutils:1.0
    command:
      - sleep
      - "infinity"
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
```

### 7 问题定位流程

首先，在dnsutils的Pod中执行`nslookup kubernetes.default`，`kubernetes.default`是default命名空间中的kubernetes服务，该服务一定会存在，如果可以正常解析该服务，说明集群的DNS解析是可以正常工作的。

如果解析`kubernetes.default`报错，则需要进一步定位：

* 检查dnsutils的Pod中的/etc/resolv.conf配置文件是否正常，例如，nameserver是否包含kube-dns这个服务的IP
* 查看CoreDNS的Pod是否运行正常，如果不正常，可以用`kubectl -n kube-system describe pod`查看Pod异常的原因，也可以查看Pod的日志是否有报错，再检查下CoreDNS的configmap中的配置是否正常
* 检查kube-system命名空间中是否存在kube-dns的Service
* 检查kube-system命名空间中kube-dns的Endpoint中是否有CoreDNS的Pod的IP
* 在CoreDNS中增加log的配置，让CoreDNS打印更加详细的日志，然后使用nslookup执行DNS查询操作，检查CoreDNS的Pod是否有接收到该请求

### 8 参考文档

* [调试 DNS 问题](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/dns-debugging-resolution/)
