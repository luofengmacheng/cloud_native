## kubernetes二次开发

kubernetes本身对很多场景已经能够达到直接使用，但是kubernetes也提供了能够扩展的能力，可以基于kubernetes进行二次开发。

[扩展kubernetes](https://kubernetes.io/zh/docs/concepts/extend-kubernetes/)

### 1 kubectl

[用插件扩展 kubectl](https://kubernetes.io/zh/docs/tasks/extend-kubectl/kubectl-plugins/)

kubectl作为k8s的客户端，它所展示的信息是固定的，有时候我们可能也需要扩展kubectl，例如，在使用网络双栈的情况下，如果直接使用kubectl只展示ipv6，需要可以同时展示ipv4和ipv6，就需要对kubectl进行扩展。

kubectl扩展的主要规则：

* kubectl扩展程序是一个在$PATH中的可执行的二进制，因此，你可以用任何语言开发插件
* 文件名以kubectl-开头，并且命令的字段之间用-分割，例如，当开发了一个插件kubectl-show展示pod的双栈地址，使用kubectl show pods时，由于没有kubectl show命令，就会去查找kubectl-show命令，找到后，就会将后面的整个字符串都传给插件程序

如果使用golang开发，已经有提供一些现成的库：

* [cli-runtime](https://github.com/kubernetes/cli-runtime)：可以用于开发插件，并且kubectl本身也使用了该库
* [sample-cli-plugin](https://github.com/kubernetes/sample-cli-plugin)：插件的简单开发示例，可以切换kubeconfig文件中指定的namespace

### 2 apiserver

* client-go：k8s的官方开发库，可以直接调用k8s的接口，常用于开发k8s的管理平台

### 3 Controller

* virtual-kubelet：虚拟kubelet，可以伪装成kubelet，常用于实现Serverless、edge IOT等
* cloud-provider(Cloud Controller Manager)：云端负载均衡器

### 4 operator(CRD & Controller)

* CRD & Controller：定义用户自己的资源，然后基于client-go开发自己的Controller控制自己资源的运行

### 5 network(CNI)

* Flannel
* Calico

### 6 store(CSI)

PV和PVC是K8S提供的让资源的提供方和资源的使用方分离的一种机制，资源提供方负责创建PV，资源使用方只需要说明申请的资源规格就能够自动创建相应规格的PV，因此，需要有程序负责自动创建PV。为了将自有的资源对接K8S，K8S提供了CSI，能够进行存储的对接。
