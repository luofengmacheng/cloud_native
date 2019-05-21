## kubernetes中的ReplicationController和ReplicaSet

### 1 存活探针

存活探针是一种探测容器是否能够正常工作(正常提供服务)的机制，其实就是`在容器外部通过某种方式访问容器，看容器是否能够正常响应`，从而判断容器是否正常。如果不正常，k8s则会重启该容器。

有三种探测的方式：

* HTTP GET 根据返回的状态码判断容器是否正常，通常用于web
* TCP 根据端口是否可以正常连接判断容器是否正常，通常用于server
* Exec 根据执行命令的返回码判断容器是否正常，不是web也不能够通过端口(或者不能完全通过端口判断)判断时

具体的使用方式可以通过查看`kubectl explain pods.spec.containers.livenessProbe`。

现在只需要知道：如果容器运行不正常，则无法从内部反馈异常信息，因此，k8s通过从外部执行某个操作的方式探测容器是否正常运行。在探测一定次数后，k8s认为容器运行不正常，则会销毁(不是重启)容器，重新创建。

### 2 ReplicationController(RC)

RC用于对pod的副本进行管理，通常，我们不会直接创建pod，而是通过更高级别的方式创建pod，RC是其中一种方式。

由于RC的配置稍微有点复杂，因此，在创建时通常都是通过yaml文件进行的。RC的yaml文件主要包含三个部分：

* 标签选择器 用于选择pod，依此判断副本数是否与配置的一致，标签选择器也可以不设置，而直接从pod模版中提取
* 副本数
* pod模版 用于创建pod的配置

``` yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-rc
  namespace: love-test
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx-rc-test
    spec:
      containers:
      - image: nginx
        name: nginx-rc-love
        ports:
        - containerPort: 80
          protocol: TCP
```

上面的yaml文件创建一个RC，RC的名字是nginx-rc，副本数是2，标签选择根据template.metadata.labels中提取，也就是app=nginx-rc-test的pod只能有两个：如果另外再创建一个pod，它的标签也是app=nginx-rc-test，那么其中一个pod会被删除。因此，RC是通过标签进行pod的托管，只要标签跟RC的标签一致就受到RC的托管。

通过yaml文件创建RC后，可以对yaml进行更改：如果更改了标签选择器，则之前创建的pod则会脱离RC的控制，请记住，RC是通过标签管理pod的，同样的，如果修改了pod的标签，则会使得pod脱离RC的控制。如果更改了模版中除标签之外的内容，不会对已经在运行的pod有任何影响，只会影响新创建的pod。

当删除RC时，相应的pod也会删除，但是要知道，pod并不算是RC的组成部分，因此，在删除RC时可以不删除pod：`kubectl delete rc nginx-rc --cascade=false`。

### 3 ReplicaSet(RS)

使用RC可以对pod进行托管，实现pod的自动扩缩容，但是RC也有自身的局限性：RC只能指定单一标签的值，也就是只能管理`app=nginx-rc-test`的pod，而不能使用多种标签。因此，k8s引入了表达力更强的RS。

