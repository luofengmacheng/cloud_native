## kubernetes网络之Flannel

### 1 Pod的IP地址的分配

当节点上只安装了docker，则会用veth pair+docker0实现单个节点上容器之间的通信，并且这些容器都在同一个IP段，如果不修改，则默认为172.17.0.0/16，此时，docker0的ip就是网段的网关地址：172.17.0.1/16。那么，多个节点上的容器都在同一个网段，跨节点通信肯定会出现问题。

因此，在k8s集群中，对于网络而言，需要有以下准则：

* 集群中的每个Pod都有独立的IP地址，且不能出现冲突
* 集群中的Pod之间可以直接通过Pod IP进行通信，且不需要做地址转换

总的来说就是，集群中的Pod都会有自己的IP地址，且能够通过这个IP地址进行通信，就好像所有Pod都在同一个网络。

因此，第一个问题就是，如何给每个Pod分配一个不冲突的IP地址。

k8s的策略是：

* 整个集群在一个大的网段中，通常是一个16位的网段，kube-controller-manager的参数--cluster-cidr就是该网段
* 每个节点从集群的网段中取一个小的网段作为这个节点上所有Pod所在的网段，通常是一个24位的网段，node.spec.podCIDR可以看到该节点的Pod网段
* 每个Pod则从节点得到的网段中获取一个IP地址

这样就保证了所有Pod的IP都在一个大的网段中，并且不会冲突。

但是，如何让Pod之间可以通过Pod IP进行通信呢？这些IP毕竟是虚拟的，需要有一种机制能够实现Pod IP的相互通信。

### 2 CNI

CNI是容器网络接口的缩写，是K8S安装Pod网络的接口。

当kubelet创建Pod时，需要调用CNI接口为Pod安装网络：

* 分配Pod的IP
* 设置Pod的网络命名空间
* 配置Pod的路由

### 3 Flannel

Flannel是最简单也比较容易理解的一个网络插件。它的主要思想是将宿主机的网络作为隧道，将Pod之间通信的网络数据包封装为可以直接在宿主机上传输的数据包。由于Pod之间传输的数据包经过了底层网络的再封装和再解封装，可以知道，这种方式存在一定的性能损耗，不过，这种损耗在大部分场景下都是可以接受的，除非对性能要求非常高。

#### 3.1 Flannel的安装

Flannel的yaml可以从官网直接下载安装，需要注意的有两个地方：

* 使用的flannel镜像如果无法下载，可以将镜像仓库修改为quay.mirrors.ustc.edu.cn
* 如果有需要可以调整Pod网段，默认的网段是10.244.0.0/16

#### 3.2 关键路径

/etc/cni/net.d/10-flannel.conflist CNI配置文件

``` json
{
  "name": "cbr0",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "flannel",
      "delegate": {
        "hairpinMode": true,
        "isDefaultGateway": true
      }
    },
    {
      "type": "portmap",
      "capabilities": {
        "portMappings": true
      }
    }
  ]
}
```

/run/flannel/subnet.env

``` shell
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.0.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
```

/opt/cni/bin flannel的二进制插件

#### 3.3 VxLan

VxLan是一项用于在三层网络传输二层网络数据包的技术，只要底层的三层网络可达，就可以在三层网络之上建立二层网络，从而构建逻辑上的大二层网络。



上图就是两个node上的Pod之间通信的流程。

* node0的网段是10.244.0.0/24，node1的网段是10.244.1.0/24
* cni0是个网桥，网桥的IP地址是网段的网关，Pod中的容器通过veth pair接入到cni0，而机器上可以看到以veth开头的网卡
* 当node0上的container1发送数据给node1上的container1时，发送端是知道目的端的IP地址的，由于目的IP不在当前网段，因此，会发送给cni0网桥
* cni0网桥收到数据后，根据路由表将数据转发给flannel.1
* flannel.1是一个vxlan的设备，VNI是1，本地地址是172.16.16.202，本地设备是eth0，目的端口是8472，也就是说，当flannel.1收到数据包后，就会将数据包封装到UDP数据包中，然后发送到对端的8472端口，问题是，flannel.1怎么知道目的IP在哪个宿主机上呢？不然，发出去的数据包如何根据宿主机网络路由到目的IP所在的宿主机呢？按照k8s的机制，是可以通过查询etcd根据目的IP的网段知道宿主机网络中的目的IP。
* 由于目的端的网络子系统会监听8472，当node1收到数据后，flannel.1就会收到数据，然后将数据解封装后，根据路由交给cni0
* cni0收到数据后，根据目的IP的mac地址转发给对应的容器

以上就完成了从源容器到目的容器的一次完整的网络数据交互。

### 3.3 UDP

### 3.4 Host-GW

host-gw是

1 veth -> cni0

2 cni0 -> flannel.1

3 flannel.1 -> dest flannel.1