## 使用luakube访问kubernetes api

### 1 kubernetes client

[客户端](https://kubernetes.io/zh-cn/docs/reference/using-api/client-libraries/)列出了各种语言对应的访问k8s的客户端库，有官方维护的，还有一些是社区和个人维护的。

这里面没有列出lua的库，在GitHub上搜索可以看到有[luakube]()，虽然好像没啥star，而且4个多月没更新了，可以用它进行测试下，如果基础的连接k8s的部分完成了，后面的各种资源操作可以自行开发和添加。

### 2 luakube初体验

要使用luakube，第一步当然是环境安装，然后执行其中的示例代码。

luakube是个lua中的包，使用luarocks进行管理，因此，先安装[lua](http://www.lua.org/download.html)和[luarocks](https://luarocks.org/)，然后再安装luakube。

luakube的依赖：

* lyaml：yaml文件的解析
* luajson：json的解析
* base64：解析token
* luasec：https
* luasocket：https依赖luasocket
* fun：提供了一些iter、map等操作，可以改写
* lpeg：luajson依赖lpeg

luarocks中每个包都有一个rockspec文件，该文件描述了该包的依赖以及一些安装信息。因此，安装luakube有两种方式：要么按照rockspec的提示一个依赖一个依赖的安装，要么直接使用rockspec文件安装：

安装方法1：

``` shell
yum install -y libyaml-devel
luarocks install lyaml
yum install -y lua-lpeg
luarocks install luajson
luarocks install luasocket
yum install -y openssl-devel
luarocks install luasec
luarocks install base64
luarocks install fun
luarocks install luakube
```

安装方法2：

``` shell
luarocks install https://raw.githubusercontent.com/f4z3r/luakube/master/rockspecs/luakube-0.1.0-0.rockspec
```

安装方法3：

``` shell
git clone https://github.com/f4z3r/luakube.git
cd luakube && luarocks make
```

第1种方法和第2种方法都是将luakube作为模块安装，相当于直接下载源码然后拷贝到安装目录，第3种方式是直接用本地目录中的rockspec文件进行安装，将本地目录的源代码拷贝到安装目录。

因此，如果是需要对luakube进行修改，可以用第3种方式，在本地目录修改后执行luarocks make，然后再进行测试。

安装完luakube，可以执行简单的demo代码进行测试，在luakube仓库的examples目录中有获取pod的logs的测试代码get_logs.lua：

``` lua
local config = require "kube.config"
local api = require "kube.api"

-- Use local kube config to connect to cluster
local conf = config.from_kube_config()
local global_client = api.Client:new(conf)

-- Get the Core V1 client
local client = global_client:corev1()

-- Get the last three lines of logs from the coredns container as a string
local container_logs = client:pods("kube-system"):logs("coredns-558bd4d5db-xr5tj", {tailLines = 3, container = "coredns"})

-- Get the logs over the last 10 seconds for all containers in the pod
local last_logs = client:pods("kube-system"):logs("coredns-558bd4d5db-xr5tj", {sinceSeconds = 10})
```

测试时需要修改上面的名称空间和pod名，就可以得到对应pod的日志。

### 3 luakube代码分析

luakube源代码分成3个部分：

* config.lua：对kubeconfig文件进行操作，外部代码实际调用时通常是调用from_kube_config(如果参数指定了kubeconfig文件，则用该文件作为凭证，如果没有指定，则用默认的kubeconfig文件路径)和in_cluster_config(在k8s集群中运行时获取sa的token作为凭证)
* api.lua：对https调用的封装
* api/：对每个version对应的资源进行操作

下面从分析examples/get_logs.lua的角度看下luakube的具体实现。

``` lua
local conf = config.from_kube_config()
```

首先获取到kubeconfig配置，如果没有提供文件名，则使用默认配置文件($HOME/.kube/config)。

``` lua
local global_client = api.Client:new(conf)
```

根据上面得到的kubeconfig配置创建client，这里创建的就是api.lua中的api.Client。

``` lua
local client = global_client:corev1()
```

调用api.lua中的api.Client:corev1()获取某个版本的api。

``` lua
local container_logs = client:pods("kube-system"):logs("coredns-558bd4d5db-xr5tj", {tailLines = 3, container = "coredns"})
```

上面的代码是获取kube-system命名空间的coredns-558bd4d5db-xr5tj这个pod的coredns容器的后面3行日志。

调用client:pods就是调用core_v1.lua中的`core_v1.Client.pods = utils.generate_object_client("pods", pod_base, true, true, true, extras)`，而utils.generate_object_client则是返回一个client，该client有各种操作资源的方法，同时还会将extras中指定的一些方法也放到该client中，对于core_v1.Client.pods，extras中有两个方法：logs和ephemeralcontainers，分别用于获取日志和临时容器。

当执行logs()时，就会调用extras中的logs方法，它会调用自身的raw_call()方法，而raw_call该方法就定义于api.lua中。

相当于调用的起点和终点都位于api.lua中。

在整体的实现中，用到了lua中的metadata编程，让返回的client拥有各种方法。

### 4 luakube包的调用

了解了上述实现的函数调用流程，在具体使用luakube包时，就可以按照k8s的设计调用相关的通用的函数。

例如，对于pod，可以使用client:pods("kube-system"):list()获取kube-system命名空间的所有pod的列表，其他的如nodes、services、configmaps、secrets、serviceaccounts等都有类似的调用。

当前资源支持的操作有：

* get(name, query)：获取某个资源，可以给出资源名称
* status(name)：资源状态
* create(obj, query)：创建资源，提供创建资源的yaml
* update(obj, query)：更新资源
* update_status(obj, query)：更新资源状态
* patch(name, patch, query, style)：变更资源
* path_status(name, patch, query, style)：变更资源状态
* delete(name, query)：删除资源
* delete_collection(body, query)
* list(query)：列出资源

其中：

* name：资源名称
* query：获取资源的条件，这个需要传table，然后会拼接到https请求的后面
* obj：资源的yaml
* patch：变更的字段
* style
* body

### 5 lua相关

#### 5.1 self

self类似于c++/java中的this和python中的self，c++/java中的this由编译器自动处理，python中的self需要开发人员添加，而lua中的self还提供了语言级别的区分：

``` lua
local t = {a = 1, b = 2}
function t:Add()
    return (self.a + self.b)
end
function t.Sub(self)
    return (self.a - self.b)
end

print(t.Add(t))
print(t:Sub())
```

如果用冒号定义和调用方法，lua解释器会自动添加self，如果用点号，则需要开发人员添加self。

#### 5.2 metadata

### 6 参考文档

* [Lua中的self](https://zhuanlan.zhihu.com/p/115159195)
* [Lua 元表(Metatable)](https://www.cnblogs.com/wwjj4811/p/14573617.html)