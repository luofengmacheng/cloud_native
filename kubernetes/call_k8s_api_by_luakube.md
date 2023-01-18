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

#### 5.2 metatable

[metatable](http://www.lua.org/manual/5.1/manual.html#2.8)翻译为元表，可以理解为table的额外属性，当访问table的一些操作时，如果table不存在，可以由元表进行完成。

lua中的metatable的概念跟python中的保留方法很像：

python中对于双下划线开头和结尾的方法有特殊含义，例如，当创建一个对象时，就是调用类的__init__()方法，当用iter某个序列时，就是调用类的__iter__()方法，也就是说，当对对象调用某个方法时，会调用某个约定好的方法，如果开发人员没有定义则调用默认的方法，当然，默认方法必须要支持此类行为，例如，如果用iter遍历一个不能遍历的对象，如果没有定义__iter__()则会调用失败。

在lua中，除了常用的基本数据类型(nil、布尔、数字、字符串)之外，table是lua提供的唯一的的复合数据类型(其他数据类型都是针对特定场景)，table可以用于实现数组、集合、字典、类等类型，而metatable就是在table上面附加的一种属性，也就是说，metatable只针对table类型。

跟python类似，在lua中，双下划线开头的方法有特殊含义，例如，当用`+`操作符对两个table相加时，就是调用第一个table的metatable中的__add()方法，这种相当于实现了对操作符的自定义，跟python中的__add__和C++中的operator +类似；当调用table一个不存在的方法时，就会调用table的metatable中的对应的方法，这种相当于实现了对类方法的动态增加。

要想使用metatable，有两个重要的方法：

* setmetatable(table, metable)：设置table的metatable，虽然这里是table，但是根据lua的文档可以知道，任何值都有metatable，但是只有table的metable可以在lua中通过setmtatable修改，其他类型只能通过C语言修改。如果metatable设置为nil，则删除table的metatable；如果table的metatable中有__metatable元素，则抛出异常，函数返回新的table
* getmetatable(object)：返回对象的metatable。如果object没有metatable，则返回nil；如果object的metatable有__metatable元素，则返回对应的值；如果object的metatable没有__metatable元素，则返回对象的metatable。

下面演示操作符的实现和方法的增加：

##### 5.2.1 使用metatable对table新增操作符

``` lua
mytable = setmetatable({ 1, 2, 3 }, {
    __add = function(mytable, newtable)
        return table.move(newtable, 1, #newtable, #mytable+1, mytable)
    end
})

secondtable = {4,5,6}

mytable = mytable + secondtable

for k,v in ipairs(mytable) do
    print(k,v)
end
```

使用setmetatable对table增加`__add`方法，`__add`实现时使用了table的move方法将第二个table中的元素放到第一个table中。

##### 5.2.2 使用metatable对table新增方法

``` lua
local mytable = {1, 2, 3}

setmetatable(mytable, {
    __index = function(mytable, key)
        return function(self, newtable)
            table.move(newtable, 1, #newtable, #self+1, self)
            return self
        end
    end
})

local secondtable = {4,5,6}

mytable:plus(secondtable)

for i=1,#mytable do
    print(mytable[i])
end
```

这里是给table增加了plus方法，通过增加带`__index`的metatable实现，`__index`对应的是个函数，而且它返回的也是个函数，返回的这个函数完成的就是两个table拼接的操作。因此，当调用mytable:plus时，由于mytable没有plus对应的值，就会查找metatable中的`__index`元素，由于`__index`存在，则会调用，并传入mytable和调用的键，因此，这里key==plus，而`__index`返回的时个函数，也就是说，mytable:plus返回的是个函数，然后用里面的这层函数调用，newtable传入的就是secondtable。

##### 5.2.3 再探luakube

luakube通过k8s的rest api访问，因此，只要知道调用接口的url和参数即可。

k8s的rest api的url格式如下：

`http://[api/apis]/api-group/api-version/namespaces/namespace/resource-kind/resource-name`

例如，获取命名空间default的所有pod：`http://127.0.0.1:8001/api/v1/namespaces/default/pods`

而k8s的rest api，其实就是完成资源操作的CRUD，只有在需要提供yaml文件时，才会把yaml文件放到body里面，其他的参数都放到url里面，因此，重点就是要对url进行拼接和调用。

luakube为了使得调用封装的调用更加灵活，将url中不同的部分放到不同的地方。

开始的api和apis在api.lua中，现在只有api/v1是api，其他都是apis。

api-group/api-version在utils.lua的utils.generate_base(api)中，其中参数api就是api-group/api-version，在下面client.call中会将self.api_拼接到路径前面进行调用。

剩下的部分则在utils.lua的utils.generate_object_client(api,concat,namespaced)中，其中参数api就是资源类型resource-kind，concat是yaml中的apiVersion和kind，对于需要yaml文件作为参数的需要，namespaced则是说明该资源是否可以指定namespace，当然，最终是否要加上namespace，还要基于是否提供namespace参数。而最后的resource-name则是通过utils.generate_object_client()中的调用方法提供。

为了将上述的调用串起来，代码中大量使用下面的操作：

``` lua
self.__index = self
setmetatable(o, self)
```

上面的代码将self作为o的metatable中的`__index`元素，于是，当访问o中的元素没有时，就会访问self中的元素，这样能够实现类似继承的机制，同时也可以给o添加方法，让o去调用self的方法，相当于子类调用父类的方法。

因此，将上面的url串起来的方式就是，在utils.generate_object_client()里面创建client，然后将core_v1.Client当作parent参数作为client的metatable，而core_v1.Client是通过utils.generate_base()返回的函数创建的client，这里又通过类似的机制调用api.lua中的方法。

总结下，通过metatable实现类似继承的机制，然后通过继承关系拼接url，最终调用api.lua中的https接口实现k8s的api的访问。

### 6 参考文档

* [Lua中的self](https://zhuanlan.zhihu.com/p/115159195)
* [Lua 元表(Metatable)](https://www.cnblogs.com/wwjj4811/p/14573617.html)
* [Lua 5.4 Reference Manual](http://www.lua.org/manual/5.4/)
* [k8s restful API 结构分析](https://www.cnblogs.com/elnino/p/9578017.html)
* [对lua继承中self.__index = self的释疑](https://www.cnblogs.com/tudas/p/how-to-understand-lua-oo-self__index.html)
* [Kubernetes API](https://kubernetes.io/zh-cn/docs/reference/kubernetes-api/)