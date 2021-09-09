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

Authorization(授权)：

授权策略(--authorization-mode)：
* AlwaysDeny：拒绝所有请求
* AlwaysAllow：允许所有请求
* ABAC(Attribute-Based Access Control)：基于属性的访问控制，表示使用用户配置的授权规则队用户请求进行匹配和控制
* Webbook：通过调用外部REST服务队用户进行授权
* RBAC(Role-Based Access Control)：基于角色的访问控制，默认策略

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
