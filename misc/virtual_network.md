## 网络虚拟化

### 1 VLAN

早期的以太网基于CSMA/CD协议，这种协议的主要问题在于广播域：所有主机属于同一个广播域，当发送广播报文时，所有主机都可以收到，这样的话，当主机较多时，会造成广播泛滥，进而影响网络传输性能。

为了减小广播域，出现了VLAN，VLAN可以将局域网划分成多个逻辑的VLAN，每个VLAN是一个广播域，不同的VLAN之间不能直接通信。

为了能够让交换机区分不同的VLAN，数据包在传输时需要携带VLAN的信息，也就是VLAN TAG，因此，需要在数据帧的头部添加一个字段：VLAN TAG。但是，VLAN通常是交换机的功能，不是所有的设备都能够处理带有TAG的数据帧，因此，交换机连接的对端的类型可以将交换机的端口进行分类：

* Access接口：对端是不能识别TAG的设备，通常是服务器。当该接口收到数据包进行转发时需要打上TAG，当该接口要发送数据包时需要删除TAG。
* Trunk接口：对端是既能够处理带TAG的设备，也能够处理不带TAG的设备，通常是交换机、路由器。
* Hybrid接口：对端既可以是服务器，也可以是交换机、路由器等，大部分场景下可以跟Trunk接口通用。

使用场景：

* 二层隔离：对用户划分不同的工作组，不同的工作组之间相互隔离，保证数据安全
* 三层互访

### 2 VxLAN

VxLAN出现的背景：

* 服务器虚拟化后，需要动态迁移，并且迁移过程中，不能影响用户正常使用
* 数据中心规模庞大，租房数量激增，需要网络提供隔离海量租户的能力

为了能够让虚拟机迁移后依然使用原来的IP地址，虚拟机必须处于一个二层网络中，反过来，如果虚拟机不是位于一个二层网络，那么迁移后就不能使用原来的IP地址吗？



### 3 ipvlan

### 4 macvlan

### 5 VPC

参考文档：

* [什么是VLAN](https://info.support.huawei.com/info-finder/encyclopedia/zh/VLAN.html)
* [VLAN基础篇](https://forum.huawei.com/enterprise/zh/forum.php?mod=viewthread&tid=246713)
* [什么是VXLAN](https://info.support.huawei.com/info-finder/encyclopedia/zh/VXLAN.html)