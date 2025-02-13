## Linux中DNS配置文件/etc/resolv.conf详解

### 1 DNS相关配置文件

在不安装额外组件的情况下，与域名解析相关的配置文件主要有三个：

* /etc/hosts：本地的域名和IP的映射关系，通常用作本地测试或者临时修改域名对应的IP
* /etc/nsswitch.conf：定义系统如何查找各种数据的方法，以及在查找时应该首先使用哪些数据库
* /etc/resolv.conf：域名解析服务器的地址以及参数配置

### 2 /etc/resolv.conf配置文件中的字段解析

/etc/resolv.conf配置文件保存的是解析域名的配置，通常包含以下配置项：

* nameserver：指定域名服务器的IP地址
* search：进行域名查询时会和域名后缀拼接成全域名去查询
* options：控制域名查询的选项

nameserver用于配置域名服务器的IP地址，例如，`nameserver 192.168.70.2`指定域名服务器的IP地址为192.168.70.2，并且可以指定多条nameserver配置，系统会按照顺序尝试去查询域名，常用的公共的域名服务器地址：

* 114.114.114.114：国内三网通
* 8.8.8.8：Google DNS

其他的可以参看[非常好用的DNS服务器](https://www.jb51.net/server/2997096vd.htm)。

search用于指定域名后缀，可以只用提供域名前缀就可以查询域名，例如，当search配置为`search baidu.com`，可以直接使用www作为域名，在进行域名查询时，会去查询`wwww.baidu.com`，而且，search后面通常也会接多个域名，那么，是不是只要查询域名就会进行拼接操作呢？这里的策略是：

* 如果提供的域名以点号结尾，则会认为是全域名，会直接查询该域名，且不会与search配置的域名后缀拼接
* 如果提供的域名中的点号的数量大于或者等于options中的ndots配置，则先直接查询该域名，如果失败，再与search配置的域名后缀拼接进行查询
* 如果提供的域名中的点号的数量小于options中的ndots配置，则直接与search配置的域名后缀拼接进行查询

例如，如果查询域名`abc.`：`host -a abc.`，会直接报错查询不到；如果查询域名`abc`：`host -a abc`，由于域名中的点号的数量等于0，小于options中的ndots配置(ndots默认值为1)，会查询`abc.baidu.com`(假设search配置为`search baidu.com`)；如果查询域名`abc.svc`，由于域名中的点号的数量为0，等于options中的ndots配置，则先查询`abc.svc`，如果失败，再查询`abc.svc.baidu.com`。

这里提到一个重要的配置项：ndots，该配置项会作为是否需要优先与search配置的域名后缀拼接还是直接查询域名的判断依据，也就是说，如果域名中的点号超过ndots，说明域名足够长，大概率是全域名，直接查询该域名，如果域名中的点号小于ndots，说明该域名比较短，大概率需要与域名后缀拼接。

options用于指定域名查询过程中的一些参数，常见的配置项有：

* timeout：设置DNS查询的超时时间
* attempts：每个DNS服务器发送查询的最大尝试次数
* ndots：控制域名查询的优先策略
* cache：控制是否使用DNS缓存

### 3 容器中的/etc/resolv.conf配置

主机上的/etc/resolv.conf配置文件用于提供域名查询的配置，在容器中当然也存在该配置，而且它还是实现服务查询的关键。

当创建普通Pod时，容器的/etc/resolv.conf中的配置是：

```
nameserver 10.1.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

其中，nameserver配置的是kube-dns服务的IP地址，search配置的是集群的域名后缀，ndots设置为5，那么，这些配置从哪里来的呢？

kubelet的启动的配置文件`--config=/var/lib/kubelet/config.yaml`中有两个参数，分别用于指定域名服务的IP和集群域名：

``` yaml
clusterDNS:
- 10.1.0.10
clusterDomain: cluster.local
```

所以，kubelet在创建容器时才可以将这些配置传入进去。

容器中服务的FQDN格式为`$SERVICE.$NAMESPACE.svc.$CLUSTER`，其中，SERVICE、NAMESPACE、CLUSTER分别表示服务名、命名空间、集群域名，而search中域名的顺序也是从小范围到大：先从相同命名空间开始查询，最后再查询整个集群。

最后一个问题：为什么ndots要设置为5呢？

当然也跟服务的FQDN格式有关，可以看到如果用全的FQDN格式，其中就有4个点号，而且，根据服务名的规范，服务名中是不能包含点号的，因此，服务的FQDN中只有4个点号。如果查询的域名超过4个点号，也就是5个或者以上，要么是集群中的服务的FQDN格式加末尾的点号，要么就不是集群中的服务，此时都可以直接向域名服务器查询，且不需要拼接；如果查询的域名不超过4个点号，那么大概率是集群中的服务，就与search中配置的域名后缀拼接后查询。
