## filebeat中增加http的output插件

### 1 缘由

官网的filebeat只有以下几种[output插件](https://www.elastic.co/guide/en/beats/filebeat/current/configuring-output.html)：

* Elasticsearch Service
* Elasticsearch
* Logstash
* Kafka
* Redis
* File
* Console

由于需要将数据推送到接口，需要能够支持类似logstash中的http插件。

### 2 编译filebeat

依照[Beats的文档](https://www.elastic.co/guide/en/beats/devguide/current/beats-contributing.html)编译filebeat：

``` shell
mkdir -p ${GOPATH}/src/github.com/elastic
git clone https://github.com/elastic/beats ${GOPATH}/src/github.com/elastic/beats
```

进入到beats根目录，执行`make mage`。

最后进入到filebeat目录，执行`mage build`。

### 3 配置虚拟机访问外网

在用golang编译时，可能需要从外部下载一些包，如果使用虚拟机，在默认情况下，宿主机可以科学上网，但是虚拟机不行，此时，可以使用宿主机作为虚拟机的代理。

* 宿主机开启代理，例如，对于Clash来说，开启"允许局域网"和"TUN模式"
* 在虚拟机中测试是否能够连接宿主机上的端口
* 配置http_proxy和https_proxy环境变量

### 4 编译beats-output-http

按照[文档](https://github.com/raboof/beats-output-http/blob/master/README.md)，直接在beats/filebeat/main.go的import部分加上`_ "github.com/raboof/beats-output-http/http"`，然后重新编译即可。

#### 4.1 使用本地包

在进行编译时，发现会报错：

```
http/client.go:82:19: assignment mismatch: 2 variables but transport.TLSDialer returns 1 value
```

应该是libbeat中的代码有变化导致的，需要修改代码，将beats-output-http克隆到本地后，需要在编译时使用本地包，可以用两种方式：

* 直接修改go.mod：在beats/go.mod中找到beats-output-http所在的行，然后在beats/go.mod文件最后增加一行`replace github.com/luofengmacheng/beats-output-http v0.0.0-20230524012217-1e747e762355 => /root/src/github.com/elastic/beats/libbeat/outputs/http`，其中，模块名和版本与之前找到的beats-output-http的行一致，后面的路径就是beats-output-http的路径（此处重命名为http）。
* 使用go mod edit命令：`go mod edit -require github.com/luofengmacheng/beats-output-http@v0.0.0-20230524012217-1e747e762355 -replace github.com/luofengmacheng/beats-output-http@v0.0.0-20230524012217-1e747e762355=/root/src/github.com/elastic/beats/libbeat/outputs/http`

这里需要注意：不要删除beats/go.mod的require中的beats-output-http的行，否则会报错：`and is replaced but not required`。

然后就可以进入filebeat目录进行编译了。

#### 4.2 发布在线包

当需要对beats-output-http进行bug修复时，可以将beats-output-http从原来的仓库fork过来，然后进行bug修复。那么，在使用beats-output-http时就需要使用新的仓库，此时需要进行两个调整：

* 修改beats-output-http/go.mod中的模块名，改成新的仓库路径
* 创建tag

然后删除beats/go.mod中原来的模块信息，再进行编译。

### 5 测试

编译完成后，会在beats/filebeat目录下生成二进制文件filebeat，创建配置文件filebeat.yaml：

``` yaml
filebeat.inputs:
    - type: log
      enabled: true
      paths:
      - /etc/kubernetes/audit/audit.log

output:
    http:
        hosts: ["IP:HOST"]
```

测试：`./filebeat run -c ./filebeat.yaml -e`。

### 6 beats-output-http的部分解释

http的output本身的实现是非常简单的，就是将数据推送到某个url，主要的工作就是要对接beats的配置和插件管理工作。

第一步：让beats知道有这样一个http插件

在libbeat的outputs包中，RegisterType()用于注册插件，其实就是将插件的名称和构造函数保存起来：

``` golang
// beats-output-http/http/http.go
func init() {
	outputs.RegisterType("http", MakeHTTP)
}
```

``` golang
// libbeat/outputs/output_reg.go
// RegisterType registers a new output type.
func RegisterType(name string, f Factory) {
        if outputReg[name] != nil {
                panic(fmt.Errorf("output type  '%v' exists already", name))
        }
        outputReg[name] = f
}
```

就是将插件的信息保存到内部的outputReg的map中，在初始化时就可以调用插件的构造函数。插件的构造函数就是解析配置，创建后端的Client对象。

第二步：Client的接口实现

filebeat调用构造函数完成初始化后，http插件就需要接收数据，然后实现具体的业务逻辑。
