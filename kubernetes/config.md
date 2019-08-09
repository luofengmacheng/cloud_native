## kubernetes中的ConfigMap和Secret

### 0 前言

应用程序启动过程中通常需要传递参数，当参数较多时会将参数进行配置化，因此，k8s提供了ConfigMap和Secret用于处理应用程序的配置。

### 1 k8s中给容器传递配置的方式

* 直接覆盖容器的ENTRYPOINT和CMD，修改容器启动的命令和参数
* 为容器设置环境变量
* 使用ConfigMap分离不同环境的配置

### 2 覆盖容器的ENTRYPOINT和CMD

``` yaml
kind: Pod
spec:
  containers:
  - image: nginx
    command: ["/bin/bash"]
    args: ["echo","test"]
```

### 3 为容器设置环境变量

``` yaml
kind: Pod
spec:
  containers:
  - image: nginx
    env:
    - name: VAR_NAME
      value: test
```

### 4 使用ConfigMap分离不同环境的配置

ConfigMap也是一种k8s的资源，将键值对的配置存放到ConfigMap中，然后在创建Pod时引用ConfigMap，从而使用ConfigMap中的配置。而且，可以在不同的环境(命名空间)中使用同名的ConfigMap，从而实现在不同的环境中使用不同的配置。

#### 4.1 创建ConfigMap

配置其实就是一个字典，因此，ConfigMap存储配置时也是以键值对的形式进行存储：

```
kubectl create configmap app-config --from-literal=environment=development --from-literal=user=admin
```

用这种方式可以直接在命令行创建ConfigMap，通过`kubectl get configmap app-config -o yaml`可以查看该配置的内容，其实就是含有两个配置项，也可以看出创建ConfigMap的yaml文件。

除了直接根据键值对创建ConfigMap，还可以从文件或者文件夹中读取：

```
kubectl create configmap app-config --from-file=filename.conf --from-file=config_item=filename.conf --from-file=config_dir
```

上面演示了三种从文件或者文件夹读取配置的方式：

* --from-file=filename.conf 会创建键为filename.conf，值为文件内容的配置项
* --from-file=config_item=filename.conf 会创建键为config_item，值为文件内容的配置项
* --from-file=config_dir 会创建以目录下文件名为键，值为对应内容的配置项

#### 4.2 使用ConfigMap

##### 4.2.1 在容器的环境变量配置从ConfigMap中读取

``` yaml
kind: Pod
spec:
  containers:
  - image: nginx
    env:
    - name: VAR_NAME
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: environment
```

##### 4.2.2 一次性将ConfigMap的所有值导入环境变量

``` yaml
kind: Pod
spec:
  containers:
  - image: nginx
    envFrom:
    - configMapRef:
        name: app-config
```

##### 4.2.3 将ConfigMap的配置项作为命令行参数

``` yaml
kind: Pod
spec:
  containers:
  - image: nginx
    env:
    - name: VAR_NAME
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: environment
    args: ["$VAR_NAME"]
```

##### 4.2.4 将ConfigMap加载为容器的卷

当配置项较多或者应用本身要使用配置文件，就需要将ConfigMap加载为容器中的卷文件，从而让应用可以直接使用配置。

如下例，将app-config的ConfigMap中的配置加载到容器中的目录/etc/nginx/conf.d。

``` yaml
spec:
  containers:
  - image: nginx
    name: test-nginx
    volumeMounts:
    - name: config
      mountPath: /etc/nginx/conf.d
      readOnly: true
  volumes:
  - name: config
    configMap:
      name: app-config
```

### 5 使用Secret分离加密配置

Secret也是配置，只是这种配置的安全性要求比较高，因此，对于需要保证安全性的配置，就需要使用Secret，例如：密码、私钥。

Secret有三种类型：

* generic 跟ConfigMap类似，直接从字符串或者文件生成，rancher中称为`密文`
* docker-registry 镜像库凭证，用于连接需要认证的docker仓库，rancher中称为`镜像库凭证`
* tls 针对https，用于存储证书，rancher中称为`证书`

#### 5.1 generic类型的Secret

使用以下命令可以创建generic的Secret：

```
kubectl create secret generic test-sec --from-literal=name=zhangsan
```

然后使用`kubectl describe secret test-sec -o yaml`查看Secret的详细信息，在结果的data部分会出现键值对，其中值是一个加密的字符串，不过它的加密很简单，就是经过base64进行简单的加密，在shell使用`base64 -d`即可进行解码。

#### 5.2 使用docker-registry类型的Secret拉取私有仓库的镜像

当访问私有仓库时，需要进行验证，例如，当访问DockerHub上的私有仓库时，需要进行登录才能拉取。

docker-registry类型的Secret就是用于访问私有仓库(需要用户名密码登录)：

```
kubectl create secret docker-registry dockersecret --docker-username=zhangsan --docker-password=zhangsanpwd --docker-email=zhangsan@google.com
```

然后在创建Pod时为Pod.spec添加imagePullSecrets字段：

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-secret
spec:
  imagePullSecrets:
  - name: dockersecret
  containers:
  - image: nginx
    name: nginx-sec
    ports:
    - containerPort: 80
      protocol: TCP
```

这样就可以拉取私有镜像，在创建Secret还能够指定`--docker-server`用于访问自己搭建的镜像仓库。

### 6 ConfigMap和Secret的对比

ConfigMap和Secret都是用于存储应用的配置，但是Secret用于存储特定场景下的安全性较高的配置。

* ConfigMap用于存放应用程序的配置，以环境变量的形式或者卷的形式进行引用
* Secret的generic类型需要进行编解码，为了保证安全性，将generic类型的Secret挂载到容器中时，是采用临时文件系统进行挂载，这样就不会将内容存储在文件中，而docker-registry和tls则是分别针对镜像仓库和https而设计的