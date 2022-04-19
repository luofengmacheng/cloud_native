## K3S具体实现

### 1 代码结构

* cmd/ 每个子目录对应k3s的一个子命令的二进制
    * agent/
    * cert/
    * containerd/
    * ctr/
    * encrypt/
    * etcdsnapshot/
    * k3s/
    * kubectl/
    * server/
* manifests/ 部分额外组件的yaml文件：ccm、coredns、local-storage、traefik
* pkg/ 每个子目录就是一个模块
    * agent/
    * apiaddressed/
    * apis/
    * authenticator/
    * bootstrap/
    * cgroups/
    * cli/ 命令的组装
    * clientaccess/
    * cloudprovider/
    * cluster/
    * codegen/
    * configfilearg/
    * containerd/ 直接调用containerd库的cmd
    * ctr/ 直接调用containerd中的cmd
    * daemons/
    * data/
    * datadir/
    * dataverify/
    * deploy/
    * etcd/
    * flock/
    * generated/
    * kubectl/ kubectl的具体实现，直接调用k8s的kubectl库的cmd
    * netutil/
    * node/
    * nodeconfig/ 给Node添加Annotation
    * nodepassword/
    * passwd/
    * rootless/
    * rootlessports/
    * secretsencrypt/
    * server/
    * servicelb/
    * static/
    * token/
    * untar/
    * util/
    * version/
* scripts/ 一些功能脚本
* tests/ 单元测试和集成测试
* main.go

main.go封装了k3s的所有子命令，每个子命令在cmd中又会生成单独的二进制程序，每个程序在实现时的调用路径是：

main(cmd/kubectl/main.go) -> flags(pkg/cli/cmds/kubectl.go 子命令的Flags和NewCommand) -> Run(pkg/cli/kubectl/kubectl.go 命令的入口函数) -> Main(pkg/kubectl/main.go 命令的具体实现逻辑)

相当于一个命令分成了两个部分：命令的组装和实现，前者负责组装整个命令，后者则负责命令的具体实现逻辑。对于kubectl而言，k3s直接使用了k8s的kubectl的客户端命令模块，因此，在pkg/kubectl/main.go中的实现主要做两件事：处理kubeconfig配置文件；执行命令command.Execute()。

k3s有以下子命令：

* server
* agent
* kubectl
* crictl
* ctr
* check-config
* etcd-snapshot
* secrets-encrypt
* certificate

其中，kubectl、crictl、ctr都是调用相关功能组件的整个包，因此，具体实现只需要看另外几个，重点肯定还是server和agent。

### 2 k3s agent

agent的实现调用跟kubectl差不多，可以从pkg/agent/run.go中的Run()开始。

* 检查ClusterCIDR、ServiceCIDR、NodeIp，如果有一个使用了IPV6，就启用IPV6
* 创建crictl的配置文件(/var/lib/rancher/k3s/agent/etc/crictl.yaml，里面存放的其实是crictl连接的端点，针对不同的操作系统有所不同)
* 如果不使用docker，并且没有提供运行时端点，则启动containerd
* 启动kubelet和kube-proxy
* 创建k8s客户端，检查/readyz是否返回ok
* 调用configureNode()配置节点
* 调用flannel库启动flannel

其中比较重要的部分其实就是启动containerd、kubelet和kube-proxy，然后再深入到这几个组件里面去会发现，它们都是使用的k8s的代码库。

pkg/daemons/executor/executor.go中定义了一个接口Executor，其中的方法就是启动各个组件，例如kubelet、kube-proxy、apiserver、scheduler等，然后在pkg/daemons/executor/embed.go中进行了实现，最后发现，其实所有这些组件都是调用k8s中对应组件的cmd。

### 3 k3s server
