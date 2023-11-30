## Operator开发之kubebuilder实战（一）

### 1 什么是Operator

k8s原生提供了很多资源类型，像用于无状态应用的Deployment、ReplicaSet和有状态应用的StatefulSet，用户可以通过向k8s提交yaml文件的方式进行资源的管理，但是，有时候，原生的这些资源无法满足我们的需求，例如，当用户希望在k8s中部署prometheus进行业务监控。

在prometheus的监控体系中，除了二进制程序，重要的就是几个配置文件：prometheus自身要抓取数据的目标、获取到数据后的检测规则等。因此，一种实现方式是将prometheus部署为Pod，并将它们的配置文件放到ConfigMap，prometheus从ConfigMap挂载，当用户修改ConfigMap时，在prometheus的Pod中加一个sidecar容器，该容器检测ConfigMap挂载的文件的变化，如果该文件变化，则调用prometheus接口让prometheus热加载配置，以此来完成配置的变更和加载操作。这种方式的问题是，对于不熟悉prometheus的用户还需要花时间去学习如何配置prometheus，而且，对于业务数据采集的场景下，prometheus的部署和维护可能是运维人员，用户通常只希望提供业务数据采集接口进行接入即可，最重要的是，这种方式不够`k8s`。

k8s提供了一种特殊的`CustomResourceDefinition资源类型`，用户可以通过这种资源类型向k8s增加新的资源类型，如上例，可以增加PrometheusScrapeTarget、PrometheusRule等资源类型，用户可以用k8s的方式创建这些资源。但是，`创建这些资源只是让k8s知道有这个新的资源类型，没有任何组件去驱动这些资源生效`，例如，当用户创建PrometehsuScrapeTarget时，在该资源中声明需要采集的Pod的selector，那么，就需要将这些Pod的IP和采集接口放到prometheus的配置文件中并加载，这就需要靠Controller。

因此，用更加k8s的方式实现上述功能，在需要告诉k8s新增资源类型的同时，还需要额外开发Controller，让新增的资源能够生效，这就是Operator，可以简单理解为`Operator = CRD + Controller`。

### 2 环境准备

#### 2.1 安装golang

在[golang](https://go.dev/dl/)下载对应平台的golang压缩包，然后解压到`/usr/local`目录下：

```shell
tar -xf go1.21.3.linux-amd64.tar.gz -C /usr/local/ 
```

然后在`/etc/profile`配置环境变量：

```shell
export GOROOT=/usr/local/go
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
export GOPROXY=https://goproxy.cn
```

然后执行`. /etc/profile`，再执行`go version`可以查看golang版本：

```shell
root@ubuntu:~# go version
go version go1.21.3 linux/amd64
```

在`/root/go_projects`目录下编写一个简单的helloworld程序测试：

```golang
// helloworld.go
package main

import "fmt"

func main()  {
	fmt.Println("Hello, World!")
}
```

直接`go run helloworld.go`可以查看输出结果。

#### 2.2 安装kubebuilder

```shell
curl -L -o kubebuilder "https://go.kubebuilder.io/dl/latest/$(go env GOOS)/$(go env GOARCH)"
chmod +x kubebuilder && mv kubebuilder /usr/local/bin/
```

然后执行`kubebuilder version`查看版本：

```shell
root@ubuntu:~/kubebuilder# kubebuilder version
Version: main.version{KubeBuilderVersion:"3.12.0", KubernetesVendor:"1.27.1", GitCommit:"b48f95cd5384eadcdfd02a47a02910f72ddc7ea8", BuildDate:"2023-09-06T06:04:11Z", GoOs:"linux", GoArch:"amd64"}
```

配置命令自动补全：

```shell
# 安装bash-completion
apt install bash-completion

echo "source <(kubebuilder completion bash)" >> /etc/profile
```

然后在使用kubebuilder过程中就可以通过tab键进行自动补全。

### 3 Operator Demo

通常一个资源的字段可能会比较多，实现起来会比较复杂，这里为了简化整个开发流程，会使用简单的例子说明。这里开发的是一个类似ReplicaSet的资源，它会负责创建一定数量的Pod，并监控Pod的状态。

#### 3.1 初始化项目

```shell
kubebuilder init --domain tutorial.kubebuilder.io --repo github.com/demo
```

* GVK：Group(api组，例如apps、batch)、Version(api组的版本，例如v1、v1beta1)、Kind(例如Deployment、CronJob)
* GVR：Group(api组，例如apps、batch)、Version(api组的版本，例如v1、v1beta1)、Resource(例如deployments、cronjobs)，大部分场景下，Kind和Resource是1对1的关系

#### 3.2 创建API

```shell
kubebuilder create api --group batch --version v1 --kind Demo
```

执行上面的命令后会自动创建以下目录和文件：

* api/v1/
      * demo_types.go Demo的类型定义
      * groupversion_info.go group和version的定义
      * zz_generated.deepcopy.go 深拷贝实现，是runtime.Object接口的自动实现
* bin/
      * controller-gen
* config/crd/
      * kustomization.yaml
      * kustomizeconfig.yaml
      * patches/cainjection_in_demos.yaml
      * patches/webhook_in_demos.yaml
* config/samples/
      * batch_v1_demo.yaml 测试用例
      * kustomization.yaml
* internal/controller/
      * demo_controller.go 控制器的调谐逻辑
      * suite_test.go 单元测试

1 api/v1/demo_types.go

该文件包含CRD的定义，Demo的结构体跟常规的yaml配置文件对应，一种资源包含4个字段：

* metav1.TypeMeta 指定GVK(apiVersion和kind)
* metav1.ObjectMeta metadata部分，包含一些公共属性
* DemoSpec 用户期望的状态
* DemoStatus 资源当前实际的状态

2 api/v1/groupversion_info.go

Group和Version的定义以及注册。

3 api/v1/zz_generated.deepcopy.go

runtime.Object接口的实现，主要是DeepCopyObject()的实现。

```golang
type Object interface {
    GetObjectKind() schema.ObjectKind
    DeepCopyObject() Object
}
```

2 internal/controller/demo_controller.go

controller的实现，里面主要的内容是Reconcile()方法：

```golang
func (r *DemoReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    _ = log.FromContext(ctx)

    // your logic here

    return ctrl.Result{}, nil
}
```

创建完成api后，就需要填充Demo的字段以及控制器的实现。

#### 3.3 填充CRD字段

这里实现Demo的简单版本，demo.spec中只有pod的模板和副本数：

```yaml
type DemoSpec struct {
       // pod模板 
	Template corev1.PodTemplateSpec `json:"template"`

       // 副本数
	Replicas *int64 `json:"replicas"`

       // pod选择器
	Selector map[string]string `json:"selector"`
}
```

而demo.status中只记录当前副本的数量：

```yaml
type DemoStatus struct {
	CurrentReplicas *int64 `json:"currentReplicas"`
}
```

其他的Demo和DemoList不做修改。接下来就需要实现控制器。

#### 3.4 实现控制器

根据Demo的字段定义以及对ReplicaSet的理解，可以设想Demo Controller的逻辑如下：

* 根据selector在Demo所在命名空间查找Pod，假设查找到N个Pod
* 如果N < replicas，则依据template模板创建replicas-N个Pod
* 如果N > replicas，则按照时间顺序删除旧的Pod
* 将N更新到demo.status.currentReplicas

首先看下kubebuilder给我们生成的脚手架代码。

跟k8s中常规代码一样，控制器的入口在`cmd/main.go`：

```golang
func main() {
    // 获取命令行参数
	var metricsAddr string
	var enableLeaderElection bool
	var probeAddr string
	flag.StringVar(&metricsAddr, "metrics-bind-address", ":8080", "The address the metric endpoint binds to.")
	flag.StringVar(&probeAddr, "health-probe-bind-address", ":8081", "The address the probe endpoint binds to.")
	flag.BoolVar(&enableLeaderElection, "leader-elect", false,
		"Enable leader election for controller manager. "+
			"Enabling this will ensure there is only one active controller manager.")
	opts := zap.Options{
		Development: true,
	}
	opts.BindFlags(flag.CommandLine)
	flag.Parse()

	ctrl.SetLogger(zap.New(zap.UseFlagOptions(&opts)))

    // 创建控制器管理器，同时传入上面的命令行参数
	mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
		Scheme:                 scheme,
		Metrics:                metricsserver.Options{BindAddress: metricsAddr},
		HealthProbeBindAddress: probeAddr,
		LeaderElection:         enableLeaderElection, // 是否启动选主逻辑，当运行多个控制器时需要开启
		LeaderElectionID:       "3d4b5da1.tutorial.kubebuilder.io",
	})
	if err != nil {
		setupLog.Error(err, "unable to start manager")
		os.Exit(1)
	}

    // 将Demo的控制器加入到控制器管理器
	if err = (&controller.DemoReconciler{
		Client: mgr.GetClient(),
		Scheme: mgr.GetScheme(),
	}).SetupWithManager(mgr); err != nil {
		setupLog.Error(err, "unable to create controller", "controller", "Demo")
		os.Exit(1)
	}
	//+kubebuilder:scaffold:builder

    // 下面两个接口用于Pod的探针配置
    // 控制器增加healthz接口
	if err := mgr.AddHealthzCheck("healthz", healthz.Ping); err != nil {
		setupLog.Error(err, "unable to set up health check")
		os.Exit(1)
	}

    // 控制器增加ready接口
	if err := mgr.AddReadyzCheck("readyz", healthz.Ping); err != nil {
		setupLog.Error(err, "unable to set up ready check")
		os.Exit(1)
	}

    // 注册SIGTERM和SIGINT信号，当收到两次信号时程序就会退出
	setupLog.Info("starting manager")
	if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
		setupLog.Error(err, "problem running manager")
		os.Exit(1)
	}
}
```

这里与我们的控制器相关的代码就是`(&controller.DemoReconciler{}).SetupWithManager(mgr)`：

```golang
// SetupWithManager sets up the controller with the Manager.
func (r *DemoReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&batchv1.Demo{}).
		Complete(r)
}
```

在Complete()中会调用Build()，这里就负责创建Controller和执行ListAndWatch操作：

```golang
func (blder *Builder) Build(r reconcile.Reconciler) (controller.Controller, error) {
        if r == nil {
                return nil, fmt.Errorf("must provide a non-nil Reconciler")
        }
        if blder.mgr == nil {
                return nil, fmt.Errorf("must provide a non-nil Manager")
        }
        if blder.forInput.err != nil {
                return nil, blder.forInput.err
        }

        // 创建Controller
        if err := blder.doController(r); err != nil {
                return nil, err
        }

        // 执行watch
        if err := blder.doWatch(); err != nil {
                return nil, err
        }

        return blder.ctrl, nil
}
```

下面就可以实现我们的控制器：

```golang
// req是变化的资源的名称，因此，在调谐逻辑开始时通常会通过名称获取对应的资源
// 返回值除了常规的error，还有一个Result，Result中包含两个元素：
// Requeue bool：表明是否重新入队列处理
// RequeueAfter time.Duration：多久之后重新入队列，如果设置了RequeueAfter则不需要设置Requeue
func (r *DemoReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	_ = log.FromContext(ctx)

	// 1 获取demo资源对象
	var demo batchv1.Demo
	if err := r.Get(ctx, req.NamespacedName, &demo); err != nil {
		log.Log.Error(err, "unable to fetch demo", "demo_name", req.Name)
		// 忽略掉 not-found 错误，通常出现在资源已经被删除时
		return ctrl.Result{}, client.IgnoreNotFound(err)
	} else {
		log.Log.Info("fetch demo success", "demo_name", req.Name)
	}

	// 2 根据资源对象获取Pod数量
	var podList corev1.PodList
	if err := r.List(ctx, &podList, client.InNamespace(req.Namespace), &client.ListOptions{
		LabelSelector: labels.SelectorFromSet(demo.Spec.Selector),
	}); err != nil {
		log.Log.Error(err, "unable to list child Pods", "demo_name", req.Name)
		return ctrl.Result{}, err
	} else {
		log.Log.Info("list child Pods success", "demo_name", req.Name, "pod_cnt", len(podList.Items))
	}

	// 3 对比当前Pod数量和期望数量
	var podCnt int64
	podCnt = int64(len(podList.Items))
	if podCnt < *demo.Spec.Replicas {
		// 当前Pod数量少于期望数量，则创建Pod
		name := fmt.Sprintf("%s-%d", req.Name, time.Now().UnixMicro())

		pod := &corev1.Pod{
			ObjectMeta: metav1.ObjectMeta{
				Labels:      make(map[string]string),
				Annotations: make(map[string]string),
				Name:        name,
				Namespace:   req.Namespace,
			},
			Spec: *demo.Spec.Template.Spec.DeepCopy(),
		}

             // 这里本来想用demo.Spec.template.ObjectMeta.Labels，发现是空的
		for k, v := range demo.Spec.Selector {
			//pod.Labels[k] = v
			pod.ObjectMeta.Labels[k] = v
		}

		if err := r.Create(ctx, pod); err != nil {
			log.Log.Error(err, "unable to create Pod", "pod_name", name)
			return ctrl.Result{}, err
		} else {
			log.Log.Info("create Pod success", "pod_name", name)
			podCnt += 1
		}
	} else {
		// 当前Pod数量大于期望数量，则需要删除多余的Pod
		// 由于可能有正在删除的Pod，因此，在进行数量判断时，需要对正在删除的Pod进行筛选
		podCnt = 0
		var pod_tmp corev1.Pod
		for _, pod_tmp = range podList.Items {
			annotations := pod_tmp.GetAnnotations()
			if annotations != nil {
				flag := false
				for k, v := range annotations {
					if k == "demo-deleting" && v == req.Name {
						flag = true
						break
					}
				}
				if flag == true {
					continue
				}
			}

			if pod_tmp.Status.Phase == corev1.PodRunning {
				podCnt += 1
			}
		}

		if podCnt > *demo.Spec.Replicas {
			// 在删除Pod之前，给Pod加一个Annotations，然后再删除Pod，后续就可以用该Annotations判断Pod是否正在删除中
			annotations := pod_tmp.GetAnnotations()
			if annotations == nil {
				annotations = make(map[string]string)
			}
			annotations["demo-deleting"] = req.Name
			pod_tmp.SetAnnotations(annotations)

			if err := r.Update(ctx, &pod_tmp, &client.UpdateOptions{}); err != nil {
				log.Log.Error(err, "update Pod annotations failed", "pod_name", pod_tmp.Name)
				return ctrl.Result{RequeueAfter: time.Second}, nil
			} else {
				log.Log.Info("update Pod annotations success", "pod_name", pod_tmp.Name)
			}

			if err := r.Delete(ctx, &pod_tmp); err != nil {
				log.Log.Error(err, "delete Pod failed", "pod_name", pod_tmp.Name)
			} else {
				log.Log.Info("delete Pod success", "pod_name", pod_tmp.Name)
				podCnt -= 1
			}

			return ctrl.Result{RequeueAfter: time.Second}, nil
		}
	}

	// 4 更新status状态
	demo.Status.CurrentReplicas = &podCnt
	if err := r.Status().Update(ctx, &demo); err != nil {
		log.Log.Error(err, "unable to update Demo status")
		return ctrl.Result{}, err
	} else {
		log.Log.Info("update Demo status success", "demo_name", demo.Name)
	}

	return ctrl.Result{}, nil
}
```

#### 3.5 部署测试

代码开发完成后，就需要进行部署测试，其实正常情况下，在未填充任何代码时也应该先部署测试，保证kubebuilder生成的代码可以在当前版本的k8s中使用。

首先看下生成的Makefile提供了哪些命令，执行`make help`可以看到支持的子命令：

开发：

* manifests 生成Webhook、ClusterRole和CRD对象，例如config/crd/bases下的CRD文件和config/crd/rbac/下面的文件
* generate 重新生成zz_generated.deepcopy.go文件
* fmt 运行go fmt格式化代码
* vet 运行go vet检查语法错误
* test 运行项目中的测试代码，主要是控制器目录的test文件

构建：

* build 构建控制器二进制
* run 在本地启动控制器
* docker-build 构建控制器镜像
* docker-push 推送控制器镜像
* docker-buildx 构建跨平台的镜像

部署：

* install 在集群中安装CRD
* uninstall 在集群中卸载CRD
* deploy 在集群中部署controller
* undeploy 在集群中卸载controller

下面就对我们开发的程序部署，跟前面说的一样，Operator主要就是CRD和Controller，部署也是一样：

* 生成CRD和RBAC：`make manifests`
* 为Demo资源重新生成zz_generated.deepcopy.go：`make generate`
* 在k8s中创建CRD：`make install`
* 本地运行Controller：`make run`

与常规的Controller不同，这种方式就会在本地启动Controller，该控制器通过~/.kube/config文件去调用k8s api的接口进行交互。

此时，可以在k8s中看到Demo资源：

![请添加图片描述](https://img-blog.csdnimg.cn/6aad5d4eb42c4d2fbc893a11f0bfe92c.png)


然后可以修改`config/samples/batch_v1_demo.yaml`文件：

```yaml
apiVersion: batch.tutorial.kubebuilder.io/v1
kind: Demo
metadata:
  labels:
    app.kubernetes.io/name: demo
    app.kubernetes.io/instance: demo-sample
    app.kubernetes.io/part-of: demo
    app.kubernetes.io/managed-by: kustomize
    app.kubernetes.io/created-by: demo
  name: demo-sample
spec:
  replicas: 2
  selector:
    app: demo-kubebuilder
  template:
    metadata:
      labels:
        app: demo-kubebuilder
    spec:
      containers:
      - name: nginx
        image: nginx
```

接下来可以将该yaml提交到k8s创建Demo对象，然后就可以查看Controller的日志：

![请添加图片描述](https://img-blog.csdnimg.cn/2f9ee97261424d6fb8e381a9af5382e4.jpeg)


从这里面可以看出`make run`其实就是`make manifests` + `make generate` + `make fmt` + `make vet` + `go run ./cmd/main.go`，因此，如果是在本地运行时如果已经将资源对象提交到k8s，后续又没有修改资源对象的字段，只是调整了Controller的代码，可以直接执行`make run`。

执行`kubectl edit demo demo-sample`修改replicas字段会发现Pod的数量也会随之变化。

#### 3.6 两个问题

对上面的代码测试会发现两个问题：

* 修改Demo的replicas字段发现Pod的数量会与期望保持一致，但是如果删除Pod，并没有新的Pod重建，这是因为当前Controller只监听了Demo的变化，没有监听Pod的变化
* Pod的删除是通过先设置Annotations，这种方式不够优雅

### 4 总结

本文用实现类似ReplicaSet的Demo资源讲解了Operator的开发过程，虽然是很简单的功能，还是需要考虑很多问题，实际的Operator开发更加复杂，考虑的问题更多，虽然k8s提供了Operator这种机制用于扩展资源，但是开发的门槛还是有些高，幸运的是，[OperatorHub](https://operatorhub.io/)提供了大量可用的Operator，在开发Operator之前可以先在这里查找下是否已经有现成的Operator可以使用。
