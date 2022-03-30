## kubernetes中的监控

### 1 监控解决方案

从实现方案来说，监控分为3个部分：数据采集、数据存储、数据分析。

数据采集是指获取采集对象的指标数据，而数据数据可以分成2种模式：推和拉。推就是agent主动将数据进行上报，拉就是服务端主动从agent拉取数据。

数据存储是指将采集的指标数据存储到

### 2 prometheus

在容器领域，提到监控就不得不提到prometheus。prometheus是一个开源的解决方案，而且可以很方便的进行扩展。

prometheus的体系中也包含上面提到的三个部分：

* exporter：负责数据采集
* prometheus：数据存储和数据分析
* alertmanager：告警推送

具体的工作流程是：exporter提供采集数据的接口，但自身并不存储数据，只是获取采集对象的数据然后格式化成指标数据，prometheus会定期从exporter拉取数据，然后将数据存储起来，prometheus自身也是个时序数据库(它最大的问题是未提供集群解决方案)，之后prometheus会定期执行用户配置的告警规则，如果满足配置的规则条件，就会调用alertmanager发送告警，alertmanager会对告警进行聚合以及执行一些抑制规则，同时，alertmanager会负责将告警发送到具体的告警通道，例如，短信、钉钉等，也可以开发alerthook程序对接用户自己的告警接口。

因此，使用prometheus监控除了需要部署prometheus以外，重要的是需要采集的对象。

#### 2.1 Node和Pod的基础监控

这部分数据包括Node和Pod的cpu、mem、network，用于kubectl top和HPA。

这部分数据的采集依旧跟上面提到的prometheus的架构类似：

* cAdvisor：负责采集数据，相当于exporter，cAdvisor当前是集成在kubelet中
* heapster/metrics-server：对指标数据进行聚合，相当于prometheus

heapster/metrics-server会在本地存放一份最近的数据，当执行kubectl top命令时，apiserver会将请求转发给heapster或者metrics-server，heapster/metrics-server则会将本地缓存的数据返回，如果给heapster/metrics-server配置了后端存储，则可以直接用grafana从后端存储中拉取数据进行监控展示。

#### 2.2 kubernetes组件自身的监控

上面的Node和Pod的指标数据主要是给kubectl top和HPA使用的，实际监控时需要监控kubernetes自身组件是否工作正常。这部分是通过kube-state-metrics采集内部对象的状态以及各组件提供的metrics接口来获取的。

#### 2.3 用户业务的监控

除了监控kubernetes组件，还需要监控用户的业务程序。如果安装了prometheus，剩下就需要对应的exporter进行谁抓取。

### 3 prometheus-operator vs kube-prometheus vs helm

使用prometheus进行监控，可以直接使用prometheus的镜像部署，将配置文件放到configmap，使用pv存储数据，但是这样做的话需要考虑prometheus上下游的组件及其容灾，因此，在kubernetes环境下，提供了operator的部署方式。

operator就是CRD+Controller，通过将prometheus中的配置抽象成kubernetes的CRD，当用户使用CRD进行部署时，Controller就会自动将用户提交的信息转换成prometheus上下游的配置，同时在信息变更时自动执行更新。

部署prometheus-operator有三种方式：

* prometheus-operator：只包含CRD+operator([bundle.yaml](https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/bundle.yaml))，但是并没有部署prometheus、exporter等组件，用户需要自行创建对应的资源进行部署。
* kube-prometheus：除了上面的CRD和operator，还会将整个监控体系都部署，例如，kube-state-metrics、node-exporter、prometheus、alertmanager。
* helm：跟kube-prometheus一样，会部署整个监控体系，只是使用了helm工具。

#### 3.1 prometheus-operator

``` shell
wget -o bundle.yaml https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/bundle.yaml
kubectl create -f bundle.yaml
```

执行上面的bundle.yaml后，会创建2部分资源：

* CRD：kubectl get crd | grep monitoring
* operator：kubectl -n monitoring get pods

CRD包含以下的资源：

* AlertManager：部署alertmanager
* PodMonitor：选择需要监控的Pod
* Prometheus：部署prometheus
* PrometheusRule：创建prometheus的监控规则
* ServiceMonitor：选择需要监控的服务
* ThanosRuler

而operator的作用就是让这些资源生效，当这些资源变更或者相关资源变更时，执行相应的变更逻辑。

所以，如果只部署上面的yaml文件，本身并没有部署任何跟监控相关的组件，例如，如果需要部署prometheus，就需要创建Prometheus资源；如果需要创建监控规则，就需要创建PrometheusRule资源。

#### 3.2 kube-prometheus

``` shell
git clone https://github.com/prometheus-operator/kube-prometheus
kubectl apply --server-side -f manifests/setup # 创建namespace和CRD
kubectl apply -f manifests/
```

上面的manifests目录中包含prometheus-operator以及整个监控体系的所有组件，包含：

* The Prometheus Operator
* Highly available Prometheus
* Highly available Alertmanager
* Prometheus node-exporter
* Prometheus Adapter for Kubernetes Metrics APIs
* kube-state-metrics
* Grafana

其中，node-exporter和kube-state-metrics负责采集数据，prometheus负责数据拉取和告警判断，alertmanager负责告警，grafana负责数据展示，可以执行`kubectl -n monitoring get pods`查看所有Pod。

#### 3.3 helm

``` shell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install [RELEASE_NAME] prometheus-community/kube-prometheus-stack
```

使用上面的命令可以直接安装整个监控体系。

### 4 

``` yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
spec:
  serviceAccountName: prometheus
  serviceMonitorSelector:
    matchLabels:
      app: monitor-app
  replicas: 2
  resources:
    requests:
      memory: 400Mi
  enableAdminAPI: false
```

![prometheus-operator ServiceMonitor](https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/Documentation/user-guides/images/architecture.png)

使用prometheus进行监控最关键的是要去配置prometheus的采集exporter的地址，而ServiceMonitor可以将用户配置的服务的信息变成prometheus的采集配置。

例如，如果我们要监控apiserver，就需要先创建一个apiserver的Service，然后创建一个ServiceMonitor，在ServiceMonitor中指定apiserver的服务的地址，

* [kubernetes-mixin](https://github.com/kubernetes-monitoring/kubernetes-mixin) Grafana dashboard和Prometheus的告警规则
* [prometheus-operator](https://github.com/prometheus-operator/prometheus-operator)

### 参考文档

* [](https://www.cnblogs.com/deny/p/13210045.html)
* [从kubectl top看K8S监控](https://www.jianshu.com/p/64230e3b6e6c)