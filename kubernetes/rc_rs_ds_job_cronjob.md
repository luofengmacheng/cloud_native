## kubernetes中托管pod的方式(ReplicationController、ReplicaSet、DaemonSet、Job、CronJob)

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

``` yaml
apiVersion: apps/v1beta2
kind: ReplicaSet
metadata:
  name: nginx-rc
  namespace: love-test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-rc-test
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

创建RS时，与RC最大的区别在于apiVersion和spec.selector字段。使用RS时，apiVersion需要设置为`apps/v1beta2`，而在spec.selector字段中添加matchLabels，因此，该RS会管理`app=nginx-rc-test`的pod。

这里暂时不对RS进行展开讲述，只需要知道：RS在标签选择器上比RC的表达力更强，通常建议使用RS而不是RC。

### 4 DaemonSet

RC和RS通常用于管理需要执行多个副本的情况，因此，常常用于部署业务的pod。而有一种特殊的场景是：需要在每个工作节点上部署一个pod，这通常会出现在监控场景，例如使用agent采集监控指标。这时候就要用到DaemonSet了。

``` yaml
apiVersion: apps/v1beta2
kind: DaemonSet
metadata:
  name: nginx-ds
  namespace: love-test
spec:
  selector:
    matchLabels:
      app: monitor
  template:
    metadata:
      labels:
        app: monitor
    spec:
      nodeSelector:
          node-type: node
      containers:
      - image: nginx
        name: nginx-ds-love
        ports:
        - containerPort: 80
          protocol: TCP
```

同样的，apiVersion也需要设置为`apps/v1beta2`，这里有三个与标签相关的配置需要注意：

* nodeSelector 节点选择器，也就是DaemonSet的pod在哪些节点上运行，这里设定只在工作节点上运行
* selector.matchLables DaemonSet管理的pod的标签
* template.metadata.labels DaemonSet创建pod时指定标签

### 5 Job

Job用于执行一次性任务，使用Job的好处在于：

* 当工作节点发生异常时，k8s会将Job调度到其它的工作节点上执行；如果Job任务的返回码非0，可以将pod配置为重新启动
* 可以在Job任务中执行多个pod，还能够定义它们之间的执行顺序

``` yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: nginx-job
  namespace: love-test
spec:
  template:
    metadata:
      labels:
        app: job-app
    spec:
      restartPolicy: OnFailure
      containers:
      - name: job-app-container
        image: once-job-image
```

创建Job时，apiVersion需要设置为`batch/v1`，特别需要注意的是job.spec.template.spec.restartPolicy：重启策略，默认值是Always，但是Job不能设置为Always，因此，该配置必须设置，可以设置为OnFailure或者Never。

### 6 CronJob

CronJob用于处理定时任务，跟linux中的crontab类似，而且时间表示方式也类似。

``` yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: nginx-cronjob
  namespace: love-test
spec:
  schedule: "*/10 * * * *"
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: cronjob-app
        spec:
          restartPolicy: OnFailure
          containers:
          - name: cronjob-app-container
            image: once-job-image
```

CronJob与Job不同的是，CronJob可以配置执行时间，配置方式跟crontab一样。

### 7 总结

k8s针对运维的不同场景设计了多种用于管理pod的方式：

* RC和RS都是用于管理需要持续运行的pod，主要用于业务的部署
* DaemonSet用于管理在每个工作节点上都需要运行的pod，主要用于监控类
* Job用于管理只需要运行一次的pod，主要用于一次性任务
* CronJob用于管理定时运行的pod，主要用于业务的定时任务