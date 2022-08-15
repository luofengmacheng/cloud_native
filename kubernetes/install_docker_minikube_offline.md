## 离线安装Docker和Minikube

### 1 Docker

从[Docker Static Binary](https://download.docker.com/linux/static/stable/x86_64/)下载docker相关二进制，将包中的二进制拷贝到/usr/bin，然后在/etc/systemd/system创建以下两个文件：

docker.service

```
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target
```

docker.socket

```
[Unit]
Description=Docker Socket for the API

[Socket]
ListenStream=/var/run/docker.sock
SocketMode=0660
SocketUser=root
SocketGroup=docker

[Install]
WantedBy=sockets.target
```

然后执行：`systemctl daemon-reload && systemctl start docker`

### 2 conntrack-tools

conntrack-tools依赖很多二进制，对于特定的操作系统，有些依赖可能已经安装，可以使用`yum install conntrack-tools --downloadonly --downloaddir=.`下载二进制，然后使用rpm命令尝试安装，安装过程中会提示缺失的依赖，缺失的依赖包可以通过repotrack命令下载。

`repotrack conntrack-tools`下载所有的依赖，当安装conntrack-tools缺少哪些依赖时可以从中拷贝。

### 3 在线安装Minikube，然后获取Binary和Image

我们先再现安装，然后从里面获取需要的二进制和镜像。

下载minikube：https://github.com/kubernetes/minikube/releases

启动minikube集群：

``` shell
minikube start --kubernetes-version=v1.21.11 --vm-driver=none --image-mirror-country=cn
```

其中，image-mirror-country设置为cn就会使用阿里云的仓库去下载二进制和镜像。

### 4 Binary

minikube start执行成功后就会在`~/.minikube/cache/linux/amd64/$K8S_VERSION`目录中出现三个二进制：kubeadm、kubectl、kubelet。

### 5 Image

minikube start执行成功后就可以使用docker images查看到使用的镜像：

```
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.21.11
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.21.11
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.21.11
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.21.11
registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.4.1
registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.13-0
registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:v1.8.0
registry.cn-hangzhou.aliyuncs.com/google_containers/storage-provisioner:v5
```

因为后续在离线安装时使用的镜像仓库是k8s.gcr.io，因此，创建新的tag：

```
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.21.11 k8s.gcr.io/kube-apiserver:v1.21.11
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.21.11 k8s.gcr.io/kube-scheduler:v1.21.11
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.21.11 k8s.gcr.io/kube-controller-manager:v1.21.11
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.21.11 k8s.gcr.io/kube-proxy:v1.21.11
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.4.1 k8s.gcr.io/pause:3.4.1
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.13-0 k8s.gcr.io/etcd:3.4.13-0
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:v1.8.0 k8s.gcr.io/coredns/coredns:v1.8.0
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/storage-provisioner:v5 gcr.io/k8s-minikube/storage-provisioner:v5
```

然后使用docker save就得到镜像的文件，就可以拷贝到新机器使用docker load导入。

### 6 Start Minikube

对于minikube来说，在新机器上安装时就可以直接将上面的4个二进制(minikube、kubeadm、kubectl、kubelet)和导出的镜像包拷贝过去，然后就可以离线启动：

``` shell
minikube start --kubernetes-version=v1.21.11 --driver=none
```

这里启动时就没有使用image-mirror-country参数，minikube就可以使用本地docker的镜像

### 7 k8s的版本问题

这里主要需要注意的是k8s的版本(`不是minikube的版本`)，网上安装minikube集群大部分的问题都是出现在网络环境，也就是下载镜像的部分，但是当前minikube默认安装k8s的版本是1.24.3，该版本有很多问题，导致在安装过程中会出现这样或者那样的问题，现在来看，v1.21.11版本安装是比较简单，需要下载的组件也比较少，像v1.24.3安装时还需要额外下载crictl和cri-dockerd，并且安装后还有各种问题。

总之，如果没有对k8s版本有硬性的要求，可以使用v1.21.11。