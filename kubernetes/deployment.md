## kubernetes中的Deployment

### 1 Why need Deployment?

K8S中Pod是用户管理工作负载的基本单位，Pod通常通过Service进行暴露，因此，通常需要管理一组Pod，前面的RC和RS主要就实现了一组Pod的管理工作，其中，RC和RS的区别在于，RS提供更加高级的选择器(RS支持[基于集合的选择符](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/labels/#%E5%9F%BA%E4%BA%8E%E9%9B%86%E5%90%88-%E7%9A%84%E9%9C%80%E6%B1%82))。

在实际生产环境中，应用在初次部署完成后，当测试正常才会对外提供服务，之后就都是升级操作，而升级过程中，意味着删除老的Pod，并创建新的Pod。对于RS而言，如果需要升级Pod的镜像，可以使用kubectl edit修改RS中的Pod的镜像，或者直接修改RS的资源文件，然后执行kubectl apply，但是，这样做的话，会使得服务在短期内的不可用。虽然Service可以保证请求只路由到ready的Pod，并且RS会按照一定的[策略](https://kubernetes.io/zh/docs/concepts/workloads/controllers/replicaset/#%E7%BC%A9%E6%94%BE-repliaset)删除老的Pod，然后再创建新的Pod，但是，在删除和创建的过程中可能无法保证服务的正常：无法保证永远有Pod在ready状态；无法保证有一定数量的Pod在处理请求。这样的话，可能会使得服务在短期内不可用。

为了解决服务更新的问题，K8S在RS的基础上提供了Deployment，它最大的特点是提供了滚动升级的能力，并且可以控制升级的速率，使得升级过程中可以正常提供服务。

### 2 Problem of ReplicaSet

设想一下，在只有RS的情况下，如何进行滚动升级，当然，在设计滚动升级流程之前必须要明确几个目标：

* 升级过程中有固定数量的Pod提供服务
* 为了保证服务不发生抖动，给定灰度的时间，可以减缓升级的速率
* 如果升级过程中发现请求处理异常可以及时回退

为了实现这些目标，在只有RS的情况下，你可能会这么做(假设RS的replicas为3)：

* 用新的Pod的template创建新的RS，然后在RS的selector中新增标签，假设为A，并且RS的replicas设置为0
* 在原来的RS的selector和Pod中都加一个标签，假设为B，这样的话，这些Pod就只被老的RS管理，并通过Service对外暴露
* 将老的RS的replicas减1，新的RS的replicas加1，现在，old_replicas=2，new_replicas=1，并且由于A、B的存在，新老RS只管理各自的Pod，互不干扰，而这些Pod都可以被Service选择
* 此时，Service的Pod同时有新老两个版本，这时候就可以通过监控查看成功率，如果没有问题，等待一段时间(相当于灰度时间)
* 确认新的Pod没有问题，继续升级：老的RS的replicas继续减1，新的RS的replicas继续加1，现在old_replicas=1，new_replicas=2，继续关注监控指标
* 继续升级：old_replicas=0，new_replicas=3，此时，所有的Pod都升级到了新版本

升级过程中总是有3个Pod提供服务，如果升级过程发现异常，可以直接修改old_replicas进行回退，同时，为了持续观察新的Pod的情况，在新的Pod创建完成后，不是立刻就对外提供服务，而是会等待一段时间，用于观察请求成功率，从而决定是否继续进行升级。

上面的过程比较复杂，而且灰度期间如果有问题，需要人工回滚，为了解决滚动更新的问题，K8S提供了Deployment，相比于ReplicaSet，它最大的优势就是滚动更新。

### 3 What is Deployment

从基础字段上看Deployment跟ReplicaSet没有任何区别，都是在Pod的基础上增加replicas字段，保证副本的数量，Deployment的主要功能也是保证副本的数量。但是，为了保证服务的可用性，相比于ReplicaSet，Deployment提供了滚动更新的能力。

Deployment提供的滚动更新跟上面的人工更新过程类似，也是创建额外的RS，然后修改新老RS的replicas值，然后再添加一些策略，例如，是全部一起更新，还是一个接一个的更新。

* 创建Deployment后，会根据
Deployment会根据pod的spec生成hash值，根据hash值

### 4 How to use Deployment

### 5 Some tips