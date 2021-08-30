## 使用kubeadm部署kubernetes集群

### 1 操作系统环境准备
* 调整内核参数：
```
echo "br_netfilter" >/etc/modules-load.d/k8s.conf
    
echo "net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
" >/etc/sysctl.d/k8s.conf
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
* 配置k8s的yum源，例如配置阿里云的源
```
echo "[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
" >/etc/yum.repos.d/kubernetes.repo
```
* containerd 容器运行时，也可以使用docker
* kubectl k8s客户端
* kubelet 监控容器运行的agent
* kubeadm 安装工具

### 3 准备镜像
* kube-apiserver
* kube-controller-manager
* kube-scheduler
* kube-proxy
* pause
* etcd
* coredns
* 非必要镜像：flannel、dashboard、metrics-scraper

### 4 使用kubeadm执行安装操作 [kubeadm init](https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm-init/)
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
* /etc/kubernetes/admin.conf