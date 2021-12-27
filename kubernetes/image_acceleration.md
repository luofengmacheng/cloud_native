## 一种镜像仓库加速的实现

### 1 客户端镜像的存储方式

#### 1.1 Docker

* /var/lib/docker/
    * containers/ 当前机器上所有的容器(docker ps -a)
    * devicemapper/
        * devicemapper/
        * metadata/
        * mnt/<容器ID>/rootfs/ 运行中的容器的根文件系统
    * image/
        * <存储驱动>/
            * repositories.json 当前机器上的镜像的列表
            * imagedb/ 镜像数据库
                * content/sha256/ 每个镜像的digest
            * layerdb/ 每个层的信息
    * network/
    * plugins/
    * swarm/
    * tmp/
    * trust/
    * volumes/

#### 1.2 Containerd

* /var/lib/containerd/
    * io.containerd.content.v1.content/blobs/sha256/ 镜像实际的数据，包含config、manifest、layer
    * io.containerd.grpc.v1.cri
    * io.containerd.metadata.v1.bolt/meta.db 使用boltdb存储镜像的元数据
    * io.containerd.runtime.v1.linux
    * io.containerd.runtime.v2.task/<namespace>/
    * io.containerd.snapshotter.v1.btrfs
    * io.containerd.snapshotter.v1.devmapper
    * io.containerd.snapshotter.v1.native
    * io.containerd.snapshotter.v1.overlayfs
    * tmpmounts
* /run/containerd/
    * containerd.sock
    * containerd.sock.ttrpc
    * io.containerd.grpc.v1.cri/
        * containers/ 所有的非pause生成的容器
        * sandboxes/ pause生成的容器
    * io.containerd.runtime.v1.linux/
    * io.containerd.runtime.v2.task/<namespace>/<container_id>/
        * config.json
        * rootfs/
        * work
    * runc/

### 2 服务端镜像的存储方式

当前服务端镜像服务的开源项目一般有2个：Registry(Docker)、Harbor(VMware)。Registry提供基础的镜像存储、上传、下载等功能，Harbor是在Registry的基础上添加了很多管理功能，更倾向于提升产品的实用性。

Registry启动时在配置文件中会指定镜像的存储目录：storage.filesystem.rootdirectory，所有的镜像都会存储在该目录。

* docker/registry/v2/
    * blobs/sha256/sha256值的前2位/sha256值/data 某个层的文件
    * repositories/
        * 仓库名称/镜像名称/
            * `_layers/sha256/sha256值/link` layers这个目录存储了该镜像包含的所有层的sha256值
            * `_uploads/`
            * `_manifests/`
                * `revisions/sha256/sha256值/link` 该镜像曾经上传过的所有版本的镜像元数据
                * `tags/标签名/`
                    * `current/link`
                    * `index/sha256/sha256值/link`

### 3 镜像操作过程

通过上面服务端镜像的存储方式，可以总结出3点：

* blobs存储所有的数据，包括manifests文件和层的文件
* repositories只存储digest信息，相当于是个索引目录

操作镜像需要有以下3个流程：

* 认证：判断客户端是合法的用户，Registry采用的认证方式是Token，当客户端(docker或者containerd)访问Registry时，如果头部没有携带认证信息，就会返回认证服务器，客户端会从认证服务器拿到token，再次发起请求
* 获取镜像的manifests：客户端提交HEAD /v2/<name>/manifests/<tag>，此时Registry就会返回`docker/registry/v2/repositories/<name>/_manifests/tags/<tag>/current/link`的内容，然后提交GET /v2/<name>/manifests/sha256:<digest>就可以返回对应的镜像的manifests文件(docker/registry/v2/blobs/sha256/<digest的前2位>/<digest>/data)，根据该文件就可以知道镜像有哪些层
* 获取层的文件：根据返回的manifests文件，解析出层的digest，然后调用GET /v2/<name>/blobs/sha256:<digest>获取对应层的文件(docker/registry/v2/blobs/sha256/<digest的前2位>/<digest>/data)

### 3 镜像加速的实现

镜像加速是容器领域非常常见的需求，通常是为了解决镜像下载速度的问题：

* 要拉取DockerHub的镜像，但是国际线路比较慢
* 公司的员工要拉取外网的镜像，由于外网带宽的问题，也比较慢

针对不同的场景，方案也有所不同：

如果是公司为了内外网分离，禁止员工访问外网，员工就只能拉取内网的镜像，那就只需要在内网搭建Registry，然后可以按天将外网的DockerHub的镜像同步到内网的Registry即可。

如果是单纯为了加速，不想每次都从外网拉取镜像，可以搭建Registry，然后将自己搭建的Registry作为缓存，然后开发一个proxy，该proxy负责接收镜像拉取请求，收到请求后如果发现本地有就直接返回，如果没有则从远程拉取。

下面详细说明下第2种方案。

registry-proxy负责截获containerd的请求，因此，需要修改containerd的配置文件，添加以下的配置：

``` toml
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
  endpoint = ["http://proxy_ip:proxy_port"]
```

上述配置会将docker.io的镜像拉取请求转发给proxy_ip:proxy_port。

当registry-proxy截获倒镜像拉取请求后，如果是获取manifests信息，由于本地可能已经不是最新的了，因此，需要从远程拉取，只有获取blobs时，才从本地拉取，毕竟blobs才是比较占用时间的请求。

### 参考文档

* [Docker Registry Api Specification](https://docs.docker.com/registry/spec/api/)
[Where are containerd’s graph drivers?](https://blog.mobyproject.org/where-are-containerds-graph-drivers-145fc9b7255)