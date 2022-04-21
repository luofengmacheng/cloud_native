## Flannel

### 1 路由表 & arp & fdb

#### 1.1 路由表

任何网络设备都需要路由表，路由表用来决定，当收到数据包时，该向哪里进行转发。路由表项通常会包含以下几个字段：

* Destination：目的地
* Gateway：网关
* Mask：掩码
* Interface：网络接口
* NextHop：下一跳

当设备收到网络数据时，从中解析出目的IP地址(假设为DEST_IP)，然后遍历Destination不是0.0.0.0的条目，并执行：`DEST_IP&Mask == Destination`，如果为真说明找到一个转发条目。于是，可以从转发条目中获取到对应的网络接口，就会将数据包从该网络接口转发出去。如果没有找到对应的转发条目，就会转发到默认网关，也就是Destination为0.0.0.0的条目。

因此，路由表是从三层的层面解决包的转发问题，Linux中通过`ip route`或者`route -n`可以查看路由表。

#### 1.2 arp表

当网络数据包转发时，底层还是要通过一个或者多个二层网络，在二层网络中就需要知道对方的MAC地址。于是，当内核的链路层收到包要进行转发时，就会去arp表查询接收方/网关的MAC地址。

arp表维护的就是ip->mac的对应关系，Linux中通过`arp -a`可以查看arp表。

#### 1.3 fdb(Forwarding DataBase)

对于二层转发来说，除了要知道ip对应的mac地址，还需要知道往哪个端口转发。fdb中保存的就是mac+vlan->port的对应关系，当收到数据包进行转发时，可以直接查看fdb转发表，决定数据包的转发端口。

执行`bridge fdb`可以查看Linux的fdb转发表。

### 2 Bridge & veth

#### 2.1 Bridge

Bridge的中文含义是网桥，提到网桥就不得不提到类似的另外3种设备：集线器、交换机、路由器，它们的主要区别是对应的网络层次不一样，能够理解的信息不一样，从而导致转发行为有所不同。

* 集线器工作在物理层，由于物理层只是处理单纯的信号，无法理解数据包的内容，因此，它在收到数据后，只能向其他所有端口转发
* 网桥工作在数据链路层，因此，网桥能够读懂数据链路帧的头部信息，也就是其中的MAC信息，因此，它在收到数据后，可以用查询MAC地址转发表，从而判断可以将数据包从哪个端口转发出去
* 交换机有二层交换机和三层交换机之分，二层交换机相当于网桥，三层交换机工作在网络层，因此，三层交换机能够读懂IP包头的信息，因此，它能够通过查询路由表，从而将数据包转发给下一跳(常见的网络拓扑结构中，接入层一般使用二层交换机，它拥有较低的成本和较多的接口数量，而汇聚层和核心层一般使用三层交换机，它拥有较高的成本和较高的转发性能)
* 路由器工作在三层，与三层交换机的主要区别在路由表项数量、转发性能上

Linux中的Bridge是个虚拟网桥，可以将网络接口加入该虚拟网桥。Linux中操作Bridge的命令有`ip bridge`、`bridge`、`brctl`。

brctl命令可以用于操作网桥：

* brctl show 查看网桥
* brctl addbr $BRNAME 创建网桥
* brctl delbr $BRNAME 删除网桥
* brctl addif $BRNAME $DEV 将接口加入网桥
* brctl delif $BRNAME $DEV 将接口从网桥中删除

剩下的命令基本都是一些配置参数相关。

安装brctl命令的方式如下：

``` shell
yum search -y bridge-utils
echo 1 > /proc/sys/net/ipv4/ip_forward
```

bridge命令有三个常用的子命令：

* bridge link 对接口进行操作，使用该命令也可以将接口加入网桥(如何删除呢？)
* bridge fdb 对fdb转发表进行操作，可以查看fdb转发表，也可以向其中插入和删除一些表项
* bridge vlan

#### 2.2 veth

veth是linux提供的一种虚拟网络接口，常用的场景是用于连接两个虚拟网络(虚拟网络设备或者网络命名空间)。

不要认为veth太过神秘，可以直接将veth理解成一根线，当数据发送给一端时，可以从另一端接收到。

因此，对veth的操作就是创建veth，然后将某一端加入某个网络环境(网络命名空间、bridge等)。

``` shell

```

上面只是简单介绍了Bridge和veth，下面可以尝试实践下。

#### 2.3 使用veth连接两个网络命名空间

这里会创建2个网络命名空间(可以理解成2个独立的网络)，然后创建一个veth，然后将veth的两端分别放到2个网络命名空间中。在操作的过程中可以注意下veth设备的命名规则。

网络命名空间的操作如下：

* ip netns add net0 创建net0的网络命名空间
* ip link set dev eth0 netns net0 将eth0加入到net0网络命名空间
* ip netns pids net0 查看net0网络命名空间中的进程
* ip netns exec net0 ping www.baidu.com 在net0网络命名空间中执行命令
* ip netns delete net0 删除net0网络命名空间

``` shell
$ip netns add net1
$ip netns add net2

$ip link add name vethtest type veth
$ip link
48: veth0@vethtest: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 9a:2f:b6:10:ae:33 brd ff:ff:ff:ff:ff:ff
49: vethtest@veth0: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 4e:f6:7e:f7:25:99 brd ff:ff:ff:ff:ff:ff
$ip link set dev veth0 netns net1
$ip link set dev vethtest netns net2
# 此时直接执行ip link则看不到veth0和vethtest两个接口，可以分别执行以下两个命令查看
$ip netns exec net1 ip link
48: veth0@if49
$ip netns exec net2 ip link
49: vethtest@if48
# 有意思的是两个接口的命名方式，两个接口后面都有一个@if加一个整数，整数刚好就是对方的序号，并且，将它们的网络命名空间改变后，这两个整数都没有变化，因此，一台服务器上面所有网络命名空间的接口序号是不会重复的，这也是判断它们属于一对veth pair的方式
# 下面分别给两个接口配置上ip地址
$ip netns exec net1 ip addr add 10.10.10.10/24 dev veth0
$ip netns exec net2 ip addr add 10.10.10.11/24 dev vethtest
#分别将两个命名空间中的lo接口和veth口的状态改为up
$ip netns exec net1 ip link set dev lo up
$ip netns exec net1 ip link set dev veth0 up
$ip netns exec net2 ip link set dev lo up
$ip netns exec net2 ip link set dev veth0test up
#此时，在两个命名空间中分别ping对方可以发现是通的
$ip netns exec net1 ping 10.10.10.11
$ip netns exec net2 ping 10.10.10.10
```

#### 2.4 使用veth连接网络命名空间和Bridge

上面的示例将veth作为两个命名空间的连接，但是实际中并不会这么使用，通常是通过veth将网络命名空间连接到Bridge，这也是容器的实现方式。

首先创建bridge，可以使用ip命令，也可以使用brctl。

``` shell
$ip link add br_test type bridge
$brctl addbr br_test

# 将网桥的状态设置为up
$ip link set dev br_test up
```

创建2个网络命名空间，表示2个容器网络：

``` shell
$ip netns add net3
$ip netns add net4
```

创建2个veth用于连接2个容器网络和bridge，并将一端加入到命名空间：

``` shell
$ip link add ctr3_dev type veth
$ip link set dev ctr3_dev netns net3
$ip link add ctr4_dev type veth
$ip link set dev ctr4_dev netns net4
```

将veth的另一端加入到bridge中，并将命名空间中的接口名修改为eth0：

``` shell
$brctl addif br_test veth0
$brctl addif br_test veth1

$ip netns exec net3 ip link set dev ctr3_dev name eth0
$ip netns exec net4 ip link set dev ctr4_dev name eth0
```

分别给2个命名空间中的接口配置IP并将状态修改为up：

``` shell
$ip netns exec net3 ip addr add 10.10.20.10/24 dev eth0
$ip netns exec net4 ip addr add 10.10.20.11/24 dev eth0

$ip netns exec net3 ip link set dev lo up
$ip netns exec net3 ip link set dev eth0 up
$ip netns exec net4 ip link set dev lo up
$ip netns exec net4 ip link set dev eth0 up

$ip link set veth0 up
$ip link set veth1 up
```

在net3中ping net4中的eth0：

``` shell
$ip netns exec net3 ping 10.10.20.11
```

### 3 VxLAN

#### 3.1 原理

VxLAN的出现是为了解决云计算时代的两个问题：

* 多租户：云环境中需要对租户进行隔离，而传统的vlan的标识符只有12位，也就是4096，完全满足不了租户数量的需求
* 虚拟机动态迁移：云环境中如果虚拟机所在宿主机异常，需要将虚拟机迁移到其他宿主机，

重要概念：

* VNI：VxLAN的标识符，占24位
* vtep：对VxLAN数据包执行封装解封装的组件，可以是硬件设备，也可以是软件设备
* mac in udp：发送方的vtep将二层的数据帧作为udp的数据进行封装，然后发送到目标的vtep，目的地的vtep收到数据后解封装，然后交给上层(因此，只要三层可达，就可以进行通信)
* 逻辑大二层网络：通过对原始的二层帧的封装解封装，在上层看来，就好像扩展了二层网络的范围，于是就称为逻辑的大二层网络(当然，不一定只有vxlan才能实现大二层网络)

#### 3.2 实验：点到点通信

在云厂商申请2台云主机，IP地址是：

* 10.23.120.82
* 10.23.72.74

我们的目的是在VPC之上创建自己的VxLAN网络，使得VxLAN网段的IP可以互通，VxLAN网段为10.10.1.0/24。

首先创建VxLAN类型的接口，然后给该接口配置上IP：

```
ip link add vxlan0(接口名称) type vxlan(网络类型) id 111(VNI) dstport 4789 remote 10.23.72.74(远程的vtep) local 10.23.120.82(本地的vtep) dev eth0
ip addr add 10.10.1.2/24 dev vxlan0
ip link set vxlan0 up
```

查看路由表：

```
# ip route
10.10.1.0/24 dev vxlan0 proto kernel scope link src 10.10.1.2
```

查看fdb表项：

```
# bridge fdb | grep vxlan0
00:00:00:00:00:00 dev vxlan0 dst 10.23.72.74 via eth0 self permanent
```

在另一台机器上也执行相同的命令，只要保证VNI一致即可。

然后我们在10.10.1.2上面执行`nc -l -p 2345`，在10.10.1.3上面执行`telnet 10.10.1.2 2345`，并在10.10.1.3上面发送数据，在10.10.1.2上也可以看到数据。在此过程中，使用tcpdump抓包。

![](https://github.com/luofengmacheng/cloud_native/blob/master/kubernetes/pics/flannel1.png)

图中可以很容易看出vxlan的mac in udp的特点：

* vxlan头部在外层的udp包中，而vxlan包的内容则是完整的tcp数据包
* vxlan头部包含4个部分：
    * Flags：标记，占1个字节
    * Group Policy ID：占1个字节
    * VNI：占3个字节，因此，VNI的范围就是`1~2^24`
    * Reserved：保留部分，占1个字节

我们来看下实际的数据通信过程：

* 当在10.10.1.3上面执行ping 10.10.1.2时，会构造icmp数据包，当到达数据链路层时，从arp表中获取10.10.1.3的mac地址，完成整个数据帧的封装
* 查看路由表(ip route)，发现10.10.1.0/24的网段需要转发给vxlan0接口(vxlan0接口就起了vtep的作用)
* vxlan0接口收到数据后，就会查看数据帧的mac地址，然后根据fdb表就知道将数据帧发送给对方的哪个vtep(`因此，fdb表里面应该有目的mac、目的vtep地址`)，这里的目的vtep就是10.23.72.74，然后将数据包封装成udp报文，目的ip就是10.23.72.74，udp的目的端口是4789，当然也会得到对应的mac地址，然后发送出去
* 目的vtep收到数据帧后，外层的udp走完完整的协议栈，会根据设定的端口信息，转发给对应的处理程序，该处理程序会获取到包中的VNI，根据VNI将数据进行转发

通过整个过程来看，VXLAN在实现中要解决的主要有三个问题：

* 由于是二层通信，那么当知道二层的ip地址时，目的mac地址如何填充(内层的mac地址的填充问题)
* vtep进行数据转发时如何知道哪个vtep是目的vtep(外层的目的ip地址如何填充)
* 如何知道哪些vtep的VNI相同呢？

第一个问题是通过发送二层的广播报文实现的，第二个问题则可以在进行arp通信时学习到该信息。

#### 3.3 实验：Bridge + VxLAN

这个例子会结合上面的VxLAN和Bridge实现类似docker的网络模式。在容器的场景下，一台物理机上面会运行多个容器，于是，就可以结合VxLAN和Bridge，将容器的网络命名空间接入Bridge，同时，将VxLAN接口也绑定到Bridge，容器之间就可以通过VxLAN网络进行通信。

在10.23.120.82上创建bridge和网络命名空间，并创建veth将网络命名空间连接到bridge：

``` shell
# 创建bridge，并启用bridge
$ip link add br_test type bridge
$ip link set dev br_test up

# 创建网络命名空间
$ip netns add n1

# 创建veth，并将一端加入bridge
# 创建veth时也可以同时制定两端的名称 ip link add veth0 type veth peer name veth1
$ip link add n1_dev type veth
$ip link set dev n1_dev netns n1
$ip link set dev veth0 up
$ip link set veth0 master br_test

# 在命名空间中设置端口的名称和状态，并在接口上配置IP(相当于容器的IP)
$ip netns exec n1 ip link set dev n1_dev name eth0
$ip netns exec n1 ip addr add 10.10.100.10/24 dev eth0
$ip netns exec n1 ip link set dev lo up
$ip netns exec n1 ip link set dev eth0 up
```

然后创建vxlan接口，并将接口绑定到bridge:

``` shell
$ip link add br_vxlan type vxlan id 333 dstport 4789 remote 10.23.72.74 local 10.23.120.82 dev eth0
$ip link set br_vxlan up
$ip link set br_vxlan master br_test
```

在另一台机器上执行类似的命名。

#### 3.4 实验：自行维护fdb

上面提过，从整个数据转发的过程来看，进行vxlan通信时只需要2部分的信息：

* 对方虚拟机或者容器的mac地址
* 对方宿主机的IP

这部分信息都在fdb转发表中维护。

因此，如果可以自行配置这部分信息，那是不是就可以直接进行VxLAN通信了。

因此，需要有一个agent程序，负责管理本宿主机上面的bridge、vxlan网卡和fdb。

### 4 CNI

### 4 Flannel