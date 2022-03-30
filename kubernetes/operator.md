## 使用operator扩展kubernetes

### 1 什么是operator？

operator是结合CRD和Controller实现对资源的控制，达到扩展kubernetes的目的。

### 2 kubebuilder(kubernetes)

#### 2.1 安装kubebuilder

`curl -L -o kubebuilder https://go.kubebuilder.io/dl/latest/$(go env GOOS)/$(go env GOARCH)`

#### 2.2 初始化项目

``` shell
mkdir cronjob && cd cronjob
kubebuilder init --repo github.com/cronjob --domain tutorial.kubebuilder.io
```

一些相关的概念：

* GVK：Group(api组，例如apps、batch)、Version(api组的版本，例如v1、v1beta1)、Kind(例如Deployment、CronJob)
* GVR：Group(api组，例如apps、batch)、Version(api组的版本，例如v1、v1beta1)、Resource(例如deployments、cronjobs)，大部分场景下，Kind和Resource是1对1的关系


* domain
* Scheme

#### 2.3 创建api

创建一个api相当于创建一个GVK：

``` shell
kubebuilder create api --group webapp --version v1 --kind CronJob
```

执行上述命令后，会自动创建以下目录：

* api/v1/
    * cronjob_types.go
    * groupversion_info.go
    * zz_generated.deepcopy.go
* bin/controller-gin
* config/
    * crd/
    * samples/
* controllers/
    * cronjob_controller.go
    * suite_test.go

1 api/v1/cronjob_types.go

该文件包含CRD的定义，CronJob的结构体跟常规的yaml配置文件对应，一种资源包含4个字段：

* metav1.TypeMeta 指定GVK(apiVersion和kind)
* metav1.ObjectMeta metadata部分，包含一些公共属性
* CronJobSpec 用户期望的状态
* CronJobStatus 资源当前实际的状态

2 api/v1/groupversion_info.go

Group和Version的定义以及注册。

3 api/v1/zz_generated.deepcopy.go

runtime.Object接口的实现，主要是DeepCopyObject()的实现。

``` golang
type Object interface {
    GetObjectKind() schema.ObjectKind
    DeepCopyObject() Object
}
```

2 controllers/cronjob_controller.go

controller的实现，里面主要的内容是Reconcile()方法：

``` golang
func (r *CronJobReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    _ = log.FromContext(ctx)

    // your logic here

    return ctrl.Result{}, nil
}
```

#### 2.4 实现api：填充Spec和Status

了解了程序的具体结构，接下来就是定义CRD，也就是填充CRD的Spec和Status部分。

#### 2.5 实现api：Reconciler

填充完Spec和Status后，就需要实现具体的逻辑，让Status达到Spec定义的期望。

这部分内容在K8S中被称为Reconciling(调协)，也即是Controller(控制器)的具体逻辑。

#### 2.5 实现api：Webhook

``` shell
kubebuilder create webhook --group webapp --version v1 --kind CronJob --defaulting --programmatic-validation
```

上述命令会创建webhook：api/v1/cronjob_webhook.go

#### 2.6 测试

``` shell
# 生成CRD资源文件
make manifests
make kustomize
bin/kustomize build config/crd > crd.yaml

# 部署CRD，然后就可以在kubectl api-resources结果中看到
kubectl apply -f crd.yaml

# 生成controller的二进制
make build

# 生成controller的镜像
make docker-build docker-push IMG=<some-registry>/<project-name>:tag

# 生成controller的部署文件
cd config/manager && ../../bin/kustomize edit set image controller=<some-registry>/<project-name>:tag
bin/kustomize build config/default > manager.yaml

# 部署controller，然后就可以看到controller-manager的Pod
kubectl apply -f manager.yaml

# 创建新的资源
kubectl create -f config/samples/batch_v1_cronjob.yaml

# 查看资源
kubectl -n cronjob-system get cronjob
```

整个部署过程中，需要注意2个地方：

1 controller的deployment中除了controller镜像外，还使用了另一个镜像gcr.io/kubebuilder/kube-rbac-proxy:v0.8.0，由于墙的原因可能无法下载，可以从[DockerHub](https://hub.docker.com/r/rancher/kube-rbac-proxy/tags)下载

2 controller在部署时使用了`runAsNonRoot: true`，针对该配置，kubelet会检查容器是否以root(uid=0)运行，如果是，则会报错。因此，要么将该配置删掉，要么多加一个配置：`runAsUser: 999`。

### 3 operator-sdk(CoreOS)