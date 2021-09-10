## kubernetes中的安全和认证

### 1 kubernetes是如何进行安全控制的？

Authentication(认证，确认双方是可信的)：
* Http Token：http header中存放Token
* Http Base：用户名和密码
* https证书：基于CA根证书签名的客户端身份认证

1 ControllerManager、Scheduler于API Server部署在同一台机器，可以直接用API Server的非安全端口访问：`--insecure-bind-address=127.0.0.1`
2 kubectl、kube-proxy访问API Server时手动颁发证书进行HTTPS双向认证
3 kubelet
4 Pod通过ServiceAccount进行访问：service-account-token

Authorization(授权，是否可以执行某项操作)：

授权策略(--authorization-mode)：
* AlwaysDeny：拒绝所有请求
* AlwaysAllow：允许所有请求
* ABAC(Attribute-Based Access Control)：基于属性的访问控制，表示使用用户配置的授权规则队用户请求进行匹配和控制
* Webbook：通过调用外部REST服务队用户进行授权
* RBAC(Role-Based Access Control)：基于角色的访问控制，默认策略

Admission(准入控制，对资源的变更加上一些逻辑限制)：

apiserver在将数据写入etcd前做一次拦截，对资源的操作进行一些逻辑控制。

### 2 RBAC

引入了4个顶级资源：
* Role
* RoleBinding
* ClusterRole
* ClusterRoleBinding

Role是资源操作权限的集合，而RoleBinding则是将User、Group、ServiceAccount与Role绑定。

Role在创建时，需要与namespace绑定，如果需要创建跨namespace的角色，可以用ClusterRole。ClusterRole通常用于：

* 集群级别的资源控制
* 非资源型endpoints
* 所有命名空间资源控制

RoleBinding不一定只能使用Role绑定，而ClusterRoleBinding只能使用在ClusterRole。另外，由于Role在创建时必须指定namespace，实际使用时不太方便，因此，通常的使用方式是：创建ClusterRole，然后使用RoleBinding将ClusterRole与用户绑定。

### 3 用户角色管理

k8s提供了4个顶级资源进行权限的授予，那么，在实际中该如何使用呢？

下面提供一种思路：

* 以项目为单位管理
* 创建普通用户的ClusterRole
* 当创建一个项目时就创建一个命名空间
* 当用户访问平台时需要添加到某个项目或者若干项目中，此时就会将用户与普通用户的ClusterRole绑定
* 后续用户就可以访问一个或者若干项目的命名空间

### 4 kubernetes默认角色

### 5 实践

#### 5.1 ServiceAccount

假设现在的场景是，我们需要创建一个用户，这个用户只能在develop这个命名空间中操作资源，SA虽然通常是Pod访问apiserver的方式，但是，也可以在外部通过SA访问集群。

```
// 创建命名空间
kubectl create namespace develop

// 创建SA(类似于创建用户)，这个SA的名命名空间是develop
kubectl create sa my-sa -n develop

// 将admin集群角色绑定到develop命名空间的SA
kubectl create rolebinding my-rb --clusterrole=admin --serviceaccount=develop:my-sa -n develop
```

通过上面的方式，my-sa这个SA就只能访问develop这个命名空间中的所有资源，那么剩下的问题就是，如果客户端要用这个SA访问该怎么办呢？那就只有为用户生成kubeconfig文件。

下面是kubeconfig的文件结构：

``` yaml
apiVersion: v1
clusters:
  - name: $CLUSTER_NAME
    cluster:
      certificate-authority-data: $CA
      server: $CLUSTER_URL
contexts:
  - name: $USERNAME
    context:
      cluster: $CLUSTER_NAME
      user: $USERNAME
      namespace: $NAMESPACE
current-context: $USERNAME
kind: Config
preferences: {}
users:
  - name: $USERNAME
    user:
      token: $TOKEN
```

从上面的文件结构来看，整个kubeconfig中有几个变量：

* 集群名称
* 集群域名
* 命名空间
* 用户名称
* 用户Token
* 用户证书

集群名称、集群域名、命名空间可以直接得到，剩下的用户名称、token和证书该如何得到呢？

SA包含三个部分：命名空间、token、证书。因此，可以直接用SA的名称、SA的token和SA的证书代替。有了上面的信息，就可以生成最终的kubeconfig文件。

#### 5.2 User

其实，通过上面的SA的例子可以看出，为了生成最终的kubeconfig，主要就是要得到一套证书和私钥。另外，为了方便kubeconfig的生成，kubernetes也提供了一些命令：

```
kubectl config set-cluster $CLUSTER_NAME --certificate-authority=公钥 --server=$CLUSTER_URL --kubeconfig=user.kubeconfig

kubectl config set-credentials --client-certificate=证书 --client-key=私钥 --username=用户名 --kubeconfig=user.kubeconfig

kubectl config set-context --cluster=$CLUSTER_NAME --user=$USERNAME --namespace=$NAMESPACE
```