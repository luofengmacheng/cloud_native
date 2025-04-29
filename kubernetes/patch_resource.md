## 对资源进行PATCH

### 1 更新资源的方式

K8S的核心就是各种资源以及针对资源的控制器，为了能够操作资源对象，apiserver提供了针对资源的CRUD的HTTP api，并且符合RESTful的规范。

创建、查询、删除相对比较简单：

* 创建使用POST方法，提交资源的yaml
* 查询使用GET方法，提供查询参数
* 删除使用DELETE方法，提供资源的名称

更新资源则有两种方式：PUT和PATCH，分别是资源的整体更新和部分更新。

* PUT是直接替换整个资源，类似于`kubectl apply -f`，如果改动的字段较多，建议用这种方式。
* PATCH是替换资源的部分字段，例如，`kubectl set image`就是直接更新资源对象的镜像。

PUT比较好理解，直接用新的yaml去覆盖老的yaml，在实际开发时，可以先GET，修改资源中的字段后，再调用PUT更新。而PATCH的部分更新可以直接修改某个字段值，例如，升级时只需要修改资源的镜像，就可以调用PATCH更新镜像，而不需要先GET再PUT。

### 2 PATCH的三种方式

PATCH虽然很方便，但是不同的使用方式适用于不同的场景。

#### 2.1 JSON Patch

通过一些操作的副词描述执行的动作：add、remove、replace、move、copy，也就是说，在对某个字段进行修改时，指定需要执行什么操作。

假设我们要修改nginx这个Deployment的镜像，则可以使用replace：

```
curl http://127.0.0.1:8001/apis/apps/v1/namespaces/default/deployments/nginx -XPATCH -H "Content-Type:application/json-patch+json" -d '[{"op":"replace","path":"/spec/template/spec/containers/0/image","value":"test-image"}]'
```

上述命令有几个关键点：

* 请求头需要带上json-patch，这样服务端才能处理这种请求
* 数据中有三个字段：op代表操作类型，path代表操作对象，value代表执行操作的值，例如，要替换的新值
* 由于containers中可能有多个容器，这里用索引定位到第一个容器

因此，当修改的内容比较多，给的数据就比较复杂，这种方式建议只修改少量字段时使用。

#### 2.2 Merge Patch

提供要修改的数据，直接将这部分数据替换进去。

同样以修改nginx这个Deployment的镜像为例：

```
curl http://127.0.0.1:8001/apis/apps/v1/namespaces/default/deployments/nginx -XPATCH -H "Content-Type:application/merge-patch+json" -d '{"spec":{"template":{"spec":{"containers":[{"name":"abc","image":"abcdef"}]}}}}'
```

* 与json patch一样，头部需要带上merge-patch
* 数据中以json格式给出要修改的数据，这里就是直接替换containers字段，无论这个Deployment原来有几个容器，执行这个命令后，都只有一个容器，所以如果要在一个Deployment中新增一个容器，就需要带上新增容器之前的所有容器配置才行

#### 2.3 Strategic Merge Patch

Strategic Merge Patch是K8S中提供的一种Patch的方式，从字面意思可以看出，这种Patch需要使用策略，可以在[API Overview](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.32/)中看到某个字段的策略：patch strategy和patch merge key，会发现，只有数组才会配置，这是因为，当使用上面的Merge Patch时，是直接采用替换的方式，但是有时候，我们又不希望增加，而是只修改其中的某个字段，或者给数组增加一个元素，此时Merge Patch就做不到了。

还是以上面的修改Deployment的镜像为例，当使用Merge Patch时，需要带上containers中的所有容器的字段值才行，如果只带上name和image，就会将现有的containers直接替换，且原有的containers下面的所有字段值都会清空。有没有一种方式，可以让我们只修改image字段，此时就可以使用Strategic Merge Patch：

```
curl http://127.0.0.1:8001/apis/apps/v1/namespaces/default/deployments/nginx -XPATCH -H "Content-Type:application/strategic-merge-patch+json" -d '{"spec":{"template":{"spec":{"containers":[{"name":"abc","image":"abcdef"}]}}}}'
```

这条命令与Merge Patch中的命令只是头部中的Content-Type不同，其他的内容完全一样，这里可以只提供要修改的容器名和镜像字段，就会直接去修改当前容器列表中name为abc的镜像为abcdef，如果没有abc这个容器，就会新增一个abc容器。

对于那些没有配置patch strategy和patch merge key的字段，使用Strategic Merge Patch跟Merge Patch一样，都是直接将字段值替换进去。

### 3 kubectl中的patch命令

上面是使用curl直接调用HTTP接口的方式，kubectl也提供patch命令用于执行patch操作。

与使用curl命令一样，在使用kubectl patch命令时，也可以选择使用Json Patch、Merge Patch、Strategic Merge Patch，默认是Strategic Merge Patch。

```
kubectl patch deployment nginx --type='merge' -p '{"spec":{"template":{"spec":{"containers":[{"name":"abc","image":"abcdef"}]}}}}'
```

上述命令其实就是将curl里面的头部换成type，而数据放到-p参数中。

如果以修改容器的镜像来看，kubectl中还有一个更简单的kubectl set image命令可以修改容器镜像：

```
kubectl set image deploy/nginx abc=test-image -v=8
```

上述命令会将nginx这个Deployment中的abc这个容器的镜像设置为test-image，这里加上-v=8可以看这个命令的详细执行过程，在执行这个命令的过程中，kubectl先是通过GET方式获取这个资源，然后通过PATCH修改镜像，这里我们只看发起的PATCH操作：

```
I0227 17:01:13.137360   22776 request.go:1181] Request Body: {"spec":{"template":{"spec":{"$setElementOrder/containers":[{"name":"abc"}],"containers":[{"image":"test-image","name":"abc"}]}}}}
I0227 17:01:13.137425   22776 round_trippers.go:463] PATCH https://172.16.16.202:6443/apis/apps/v1/namespaces/default/deployments/nginx?fieldManager=kubectl-set
I0227 17:01:13.137432   22776 round_trippers.go:469] Request Headers:
I0227 17:01:13.137441   22776 round_trippers.go:473]     Accept: application/json, */*
I0227 17:01:13.137449   22776 round_trippers.go:473]     Content-Type: application/strategic-merge-patch+json
```

可以看到，set image也是用的Strategic Merge Patch，唯一与我们自己的curl调用有区别的是：

* 请求体中除了给containers，还有个`$setElementOrder/containers`字段，该字段是K8S内部用于确保数组中的字段值的顺序在修改过程中不会被意外修改
* 请求的url中带上了fieldManager字段，该字段是K8S内部用于跟踪字段修改的机制，这里表示是kubectl set命令修改的

### 4 PATCH的优势和问题

与UPDATE相比，PATCH可以用于只修改某几个字段的场景，而且如果数据没有问题，PATCH不会失败，而UPDATE可能存在并发的问题。

使用PATCH时，需要详细了解Json Patch、Merge Patch、Strategic Merge Patch的区别和使用场景，如果使用的不对，也可能造成其他问题。

一般来说，通常建议使用Strategic Merge Patch，其次是Json Patch。

### 5 参考文档

* [JSON Patch and JSON Merge Patch](https://erosb.github.io/post/json-patch-vs-merge-patch/)
* [使用 kubectl patch 更新 API 对象](https://kubernetes.io/zh-cn/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/)
* [kubectl patch](https://kubernetes.io/zh-cn/docs/reference/kubectl/generated/kubectl_patch/)
