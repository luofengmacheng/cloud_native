## 使用kubeadm部署kubernetes集群

### 1 操作系统环境准备
* 调整内核参数：
```
modprobe br_netfilter

sysctl -w net.ipv4.ip_forward=1
sysctl -w net.bridge.bridge-nf-call-ip6tables=1
sysctl -w net.bridge.bridge-nf-call-iptables=1
```
* 禁用SELINUX
```
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```
* 关闭swap
```
swapoff -a
sed -i /swap/d /etc/fstab
```

### 2 安装containerd、kubectl、kubeadm、kubelet

#### 2.1 安装containerd

``` shell
wget https://github.com/containerd/containerd/releases/download/v1.5.7/containerd-1.5.7-linux-amd64.tar.gz
tar xvf containerd-1.5.7-linux-amd64.tar.gz
cp -r bin/* /usr/local/bin/
containerd config default > /etc/containerd/config.toml
# 将https://github.com/containerd/containerd/blob/main/containerd.service文件拷贝到/usr/lib/systemd/system/
systemctl start containerd
```

#### 2.2 配置yum源

* 配置k8s的yum源，例如配置阿里云的源
```
echo "[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
" >/etc/yum.repos.d/kubernetes.repo
```

* 安装kubectl、kubelet、kubeadm
``` shell
yum -y install kubectl kubelet kubeadm
```

```
yum install -y libseccomp cri-tools kubeadm-$k8s_ver kubectl-$k8s_ver kubelet-$k8s_ver --disableexcludes=kubernetes
```

### 3 准备镜像
* kube-apiserver
* kube-controller-manager
* kube-scheduler
* kube-proxy
* pause
* etcd
* coredns
* 非必要镜像：flannel、dashboard、metrics-scraper

从阿里云上下载镜像，执行`kubeadm config images list`可以知道需要哪些镜像，然后将这些镜像拉取到k8s.io空间中：

``` shell
K8S_VERSION="v1.22.3"
ctr -n k8s.io image pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver-amd64:$K8S_VERSION
ctr -n k8s.io image pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:$K8S_VERSION
ctr -n k8s.io image pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:$K8S_VERSION
ctr -n k8s.io image pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:$K8S_VERSION
ctr -n k8s.io images pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2
ctr -n k8s.io images pull registry.cn-hangzhou.aliyuncs.com/openthings/k8s-gcr-io-coredns:1.2.6
ctr -n k8s.io images pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.5.0-0
```

### 4 使用kubeadm执行安装操作 [kubeadm init](https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm-init/)

``` shell
kubeadm init --kubernetes-version v1.22.3 --image-repository registry.aliyuncs.com/google_containers --pod-network-cidr 10.244.0.0/16
```

* `kubeadm init phase certs all` 创建证书
* `kubeadm init phase kubeconfig all` 为admin、controller-manager、kubelet、scheduler生成kubeconfig配置文件
* `kubeadm init phase kubelet-start` 启动kubelet
* `kubeadm init phase control-plane all` 为apiserver、controller-manager、scheduler生成Pod描述文件
* `kubeadm init phase etcd local` 生成单节点的etcd的Pod描述文件
* `kubeadm init phase upload-config all` 将kubeadm的集群配置文件和kubelet的配置文件上传为configmap
* `kubeadm init phase upload-certs --upload-certs` 上传证书
* `kubeadm init phase mark-control-plane` 将节点标志为控制节点并打标
* `kubeadm init phase bootstrap-token`
* `kubeadm init phase kubelet-finalize all` 在TLS引导后更新kubelet相关的配置
* `kubeadm init phase addon all` 安装额外的组件，coredns、kube-proxy

### 5 保存集群配置，用于后续集群访问
* mkdir ~/.kube && cp /etc/kubernetes/admin.conf ~/.kube/kubeconfig

### 6 部署网络插件(flannel)

* 部署flannel时需要注意在执行kubeadm时需要带上`--pod-network-cidr`参数
* `kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml`

### 7 部署Node节点

* Node节点只需要部署containerd、kubelet、kubeadm，同时还需要pause镜像
* kube-proxy不需要部署，由containerd-shim-runc-v2负责启动(如何下载程序的暂时未知)
* 执行以下命令加入集群，`kubeadm join $APISERVER_ADDRESS --token $TOKEN --discovery-token-ca-cert-hash $CA_CERT_HASH`，当Master节点部署完成后会给出该命令

### 8 自签名CA证书

``` shell
mkdir -p /etc/uec
pushd /etc/uec

# 1 生成根证书
# 生成根证书的私钥
openssl genrsa -out cakey.pem
# 生成自签名证书时需要带上-x509参数
openssl req -new -x509 -key cakey.pem -out cacert.pem -subj '/CN=Mirror CA/O=xxx/ST=xxx/L=xxx/C=CN' -days 3650
cat cacert.pem >>/etc/pki/tls/certs/ca-bundle.crt

# 2 生成私钥和证书请求文件
openssl genrsa -out k8s.io.key 2048
openssl req -new -key k8s.io.key -out k8s.io.csr -subj '/CN=*.k8s.io/O=xxx/ST=xxx/L=xxx/C=CN'

# 3 颁发证书
openssl x509 -req -in k8s.io.csr -CA cacert.pem -CAkey cakey.pem -CAcreateserial -out k8s.io.crt -days 3650 -config ./openssl.cfg -extensions k8s.io
```

### 9 高可用

高可用根据业务类型分为2种实现：

* 多个业务程序可以同时运行，都可以对外提供服务
* 多个业务程序可以同时运行，但只有一个可以对外提供服务

对于运行在kubernetes中的程序，第一种方式通过多副本的RS/Deployment进行部署，第二种方式则要依赖额外的组件提供。

而对于kubernetes自身的高可用，则需要分别对master上运行的组件进行处理。

* kube-scheduler：调度器的功能是对未调度的Pod进行调度，如果多个调度器同时运行，会发生冲突，因此，调度器就属于上面的第二种情况
* kube-controller-manager：控制器管理器是执行具体的调谐逻辑，跟调度器一样，属于上面的第二种情况
* etcd：etcd本身就是分布式的键值对存储系统，可以部署多个实例进行容灾

对于kube-scheduler和kube-controller-manager而言，提供了`--leader-elect`选项，该选项默认为true，与该选项相关的其他参数有：

* --leader-elect-lease-duration duration，默认值为15s，非领导者身份的节点，如果发现领导者身份有变化，需要等待一段时间才能尝试去获取领导者身份
* --leader-elect-renew-deadline duration，默认值为10s，领导者身份的节点，每隔一段时间更新
* --leader-elect-resource-lock string，默认值为leases，领导者选举期间用于锁定的资源对象的类型
* --leader-elect-resource-name string，默认值为kube-controller-manager(对于controller-manager而言)，用于锁操作的资源对象名称
* --leader-elect-resource-namespace string，默认值为kube-system，用于锁操作的资源对象的命名空间
* --leader-elect-retry-period duration，默认值为2s，尝试获得领导者身份时，相邻两次尝试之间要等待的时长

从上面的一些选项就可以看出领导者选举的具体实现方案：

* 程序启动后，根据本机的主机名和uuid(uuid.NewUUID)生成id
* 如果leader-elect为false，则直接执行逻辑
* 如果leader-elect选项为true或者未配置，则进入领导者选举阶段
* 每隔leader-elect-retry-period时间就将id写入leader-elect-resource-namespace命名空间中的leader-elect-resource-lock资源，资源名称为leader-elect-resource-name，如果写成功，就成为领导者
* 领导者每隔leader-elect-renew-deadline时间刷新资源更新时间
* 非领导者如果发现超过leader-elect-lease-duration时间未续期，则进行领导者抢占

对于kube-apiserver而言，它是无状态的，可以部署多个实例，可以同时运行，但是，Node连接时只能指定一个IP，因此，可以部署多个kube-apiserver实例，然后使用nginx+keepalived+VIP对外暴露一个IP。而Node在加入集群时，则直接指定VIP即可。