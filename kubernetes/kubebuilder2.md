## Operator开发之kubebuilder实战（二）

### 1 解决（一）中的两个问题

在[Operator开发之kubebuilder实战（一）]()的后面提出了两个问题：

* 由于DemoController只监听了Demo资源的变化，因此，删除Pod时，DemoController不会创建Pod，同时，在删除Demo时，也不会删除对应的Pod，因为此时已经没有Demo了，而且k8s也不知道该Demo管理了哪些Pod
* 为了在调谐逻辑中过滤掉正在执行删除的Pod，在删除Pod之前会加一个Annotation，这种处理方式不够优雅

首先，DemoController需要监听Pod资源的变化，kubebuilder提供了两种监听额外资源的变化的机制，一种是简单粗暴的`Watchs`：

``` golang
func (r *DemoReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&batchv1.Demo{}).
		Watches(&corev1.Pod{}, &handler.EnqueueRequestForObject{}).
		Complete(r)
}
```

这种方式就可以直接ListAndWatch Pod资源，这种方式的问题是，可以接收到`所有命名空间的Pod的变化`，但是，在我们的场景中，希望只监听Demo负责创建的Pod。

因此，另一种就是`Owns`：

``` golang
func (r *DemoReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&batchv1.Demo{}).
		Owns(&corev1.Pod{}).
		Complete(r)
}
```

这种方式就只会监听Demo负责创建的Pod，不过使用这种方式还需要在创建Pod时增加ownerReferences属性，这种方式就跟ReplicaSet类似：当ReplicaSet在创建Pod时，会将自身的信息写入到Pod的ownerReferences字段，当Pod变化时，会根据Pod找到关联的ReplicaSet，然后向ReplicaSet推送Pod变更。这种方式可以让DemoController接收必要的消息，而不需要实现过滤逻辑。

创建Pod时在metadata部分加入ownerReferences：

``` golang
        pod := &corev1.Pod{
			ObjectMeta: metav1.ObjectMeta{
				Labels:      make(map[string]string),
				Annotations: make(map[string]string),
				Name:        name,
				Namespace:   req.Namespace,
				OwnerReferences: []metav1.OwnerReference{
					{
						APIVersion:         demo.APIVersion,
						Kind:               demo.Kind,
						Name:               demo.Name,
						UID:                demo.UID,
						Controller:         pointer.Bool(true),  // 为true表示该资源是控制器
					},
				},
			},
			Spec: *demo.Spec.Template.Spec.DeepCopy(),
		}
```

通过这两个改动的结合就可以达到效果：

* 通过监控Pod的变化，能够在Pod被删除时去判断当前的Pod数量和需要的Pod数量，就可以保证在Pod删除时进行Pod的重建
* 通过给Pod设置ownerReferences属性，能够将Pod与Demo关联起来，一方面，在Pod变更时能够找到关联的Demo并给Demo推送变更，另一方面，在删除Demo时，可以级联删除Pod

需要注意的是：

* 在`SetupWithManager`里面监听的是Pod，但是调谐函数收到的是Pod关联的Demo的变更，而不像有些文章写的在这里还需要判断收到的变更是Demo还是Pod
* 删除Demo时，会级联删除Pod

关于`级联删除`：当删除一个资源时，该如何删除这个资源管理的资源。当使用`kubectl delete`删除资源时，有个级联删除的选项`--cascade`，该选项有三个值：

* background：后台级联删除，是默认的删除策略，k8s立即删除当前资源，然后在后台清理当前资源创建的其他资源（也就是ownerReferences是当前资源的资源）
* foreground：前台级联删除，kubectl会，k8s会给Demo加上deletionTimestamp属性和finalizers属性，其中finalizers设置为`foregroundDeletion`，给Pod设置deletionTimestamp，然后去删除Pod，当清理操作完成时，再移除finalizers属性并删除Pod
* orphan：不进行级联删除，直接删除当前资源，也就是不管当前资源创建的其他资源，跟我们没有用ownerReferences效果一样

这就解释了为什么给Pod加上ownerReferences属性就可以在删除Demo时自动删除Pod。

资源删除过程中还需要提到两个属性：`BlockOwnerDeletion`和`Finalizers`：

* BlockOwnerDeletion：
* Finalizers：当资源设置了Finalizers时，k8s收到删除请求时，不会直接删除资源，而是会添加资源的deletionTimestamp属性，然后返回Accepted，之后FinalizersController会监听该资源，当该资源的条件满足时，就会从



### 2 开发Webhook

#### 2.1 为什么需要Webhook

#### 2.2 基于kubebuilder开发Webhook

### 3 
