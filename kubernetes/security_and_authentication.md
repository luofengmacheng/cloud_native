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

如果是通过TOKEN的方式认证：

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

如果是通过https证书认证的方式：

``` yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: $CERTIFICATE_AUTHORITY_DATA
    server: $CLUSTER_URL
  name: default
contexts:
- context:
    cluster: default
    user: default
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: $USERNAME
  user:
    client-certificate-data: $CLIENT_CERTIFICATE_DATA
    client-key-data: $CLIENT_KEY_DATA
```

* CERTIFICATE_AUTHORITY_DATA：CA证书，用于验证服务端发送来的证书
* CLIENT_CERTIFICATE_DATA：客户端证书，发送给服务端，从中提取出公钥
* CLIENT_KEY_DATA：客户端私钥

这种方式的认证流程就是：

* 客户端连接server中指定的集群地址
* 服务端返回证书，客户端用指定的CA证书对服务端返回的证书进行验证，并从中提取出服务端的公钥
* 客户端将自己的证书发送给服务端
* 服务端同样对客户端发送的证书进行验证，并从中提取出客户端的公钥
* 客户端生成随机数，使用服务端的公钥进行加密
* 服务端收到数据后，使用私钥进行解密，然后提取出随机数
* 客户端与服务端使用随机数作为对称密钥进行通信

因此，使用https相比于token的好处就是：token只验证了服务端，而https会对服务端和客户端都进行验证。

#### 5.2 User

其实，通过上面的SA的例子可以看出，为了生成最终的kubeconfig，主要就是要得到一套证书和私钥。另外，为了方便kubeconfig的生成，kubernetes也提供了一些命令：

```
kubectl config set-cluster $CLUSTER_NAME --certificate-authority=公钥 --server=$CLUSTER_URL --kubeconfig=user.kubeconfig

kubectl config set-credentials --client-certificate=证书 --client-key=私钥 --username=用户名 --kubeconfig=user.kubeconfig

kubectl config set-context --cluster=$CLUSTER_NAME --user=$USERNAME --namespace=$NAMESPACE
```

### 6 公钥、私钥、签名、证书

```
加密是为了保证数据的安全性
签名是为了保证数据的完整性
证书是为了保证公钥的完整性
```

加密一般分为对称加密和非对称加密：

* 对称加密指的是通信的双方用`相同的密钥`对数据进行加解密：明文+密钥->密文，密文+密钥->明文
* 非对称加密指的是通信的双方用`不同的密钥`对数据进行加解密：用公钥加密的密文只能用私钥解密，用私钥加密的密文只能用公钥解密

对称加密的好处在于简单并且效率高，不好的当然就是密钥的分发问题。非对称加密的好处在于安全性高，不好的就是加解密效率比对称加密低。

签名就是对通信的数据打上标签，可以防止数据被篡改以及保证数据的真实性。

生成签名的过程：

* 对数据计算HASH值
* 用私钥对HASH值加密，生成签名
* 将签名放在消息后面一起发送

验证签名的过程：

* 用公钥对数据中的签名进行解密，得到HASH1
* 对数据中的正文计算哈希值，得到HASH2
* 比较HASH1和HASH2，如果相同，则验证成功

生成证书的过程：

* 服务器将公钥A给CA
* CA用自己的私钥B对公钥A加密，生成数字签名
* CA把公钥A、数字签名、服务器信息整合生成证书
* CA将证书分发给服务器

通过上面生成证书的过程可以知道，证书主要包含两部分信息：自己的公钥和数字签名(只有用CA的公钥才能对数字签名进行验证)。

验证证书的过程：

* 当客户端与服务端通信时，服务端将自己的证书发给客户端
* 客户端收到服务端发送的证书，需要从证书中得到服务端的公钥。为了得到公钥，又需要验证公钥没有被篡改，就只能用签发证书的CA的公钥进行验证。
* 假设客户端已经有签发证书的CA的公钥，用该公钥对证书进行解密并计算哈希值，然后与证书中的公钥计算的哈希值进行对比，如果相同，说明公钥没有被篡改
* 客户端就得到了服务端的公钥，然后客户端在发送数据时就使用服务端的公钥，服务端返回收据时就使用服务端的私钥

根证书：

上面验证证书的过程中，有个问题没有解答：客户端如何得到签发证书的CA的公钥？

举个例子，当访问`www.baidu.com`时，返回了百度的证书，为了得到并验证百度的公钥，又必须有签发给百度证书的GlobalSign Organization Validation CA机构的公钥，因此，客户端又必须有GlobalSign Organization Validation CA机构的证书，从而可以从中提取出百度的公钥。但是，怎么去验证GlobalSign Organization Validation CA的证书呢？又必须有签发给GlobalSign Organization Validation CA的证书公钥，即GlobalSign Root CA的公钥。这就构成了一个信任链，那么这个链在哪里结束呢？就是根证书，根证书是由权威机构给自己颁发的证书。

当前，权威的颁发根证书的机构并不多，可以认为，电脑中已经默认保存了大部分的权威机构的根证书，也就是这些证书是可以信任的。然后再用这些根证书去验证下面的机构或者网站的证书。

### 7 HTTPS

前面提到过对称加密和非对称加密的优缺点，而HTTPS就刚好结合了两者的优点：

* 对称加密的高效性
* 非对称加密的安全性

#### 7.1 单向认证

单向认证就是我们通常访问网站的方式：只验证服务端是否是安全的。

认证流程：

* 客户端建立HTTPS请求
* 服务端返回证书
* 客户端通过上面的验证流程从证书中提取并验证服务端的公钥
* 客户端生成随机数R，用服务端的公钥加密，发送给服务端
* 服务端使用私钥解密得到随机数R
* 客户端和服务端使用随机数R作为对称密钥进行通信

上述过程只验证服务端是否安全，而没有验证客户端。

#### 7.2 双向认证

双向认证既需要验证服务端，又需要验证客户端。单向认证只为服务端生成了私钥和证书，而双向认证就需要为客户端和服务端分别生成私钥和证书。

认证流程：

* 客户端建立HTTPS请求
* 服务端返回证书server.crt
* 客户端验证证书并从中提取出服务端的公钥server.pub
* 客户端将自己的证书client.crt发送给服务端
* 服务端验证证书并从中提取出客户端的公钥client.pub
* 客户端生成随机数R，使用服务端公钥加密后发送给服务端
* 服务端使用私钥解密得到随机数R
* 客户端和服务端使用随机数R作为对称密钥进行通信

### 8 kubernetes中的证书

kubernetes中的所有组件通信时都采用HTTPS双向认证，而部署时不可能为他们申请证书，因此，在部署kuberentes时通常都是先生成自签名证书，然后用该证书作为根证书，后续的证书都通过它签发。

``` shell
mkdir -p /etc/crt
pushd /etc/crt

# 使用RSA算法生成私钥
openssl genrsa -out cakey.pem

# 根据私钥得到公钥，然后生成证书
# -new：生成新的证书请求文件
# -x509：结合-new表示生成证书文件
# -key：私钥
# -subj：机构信息(网站信息)
# -days：有效期
openssl req -new -x509 -key cakey.pem -out cacert.pem -subj '/CN=Mirror CA/O=UCloud/ST=GuangDong/L=Shenzhen/C=CN' -days 3650
cat cacert.pem >>/etc/pki/tls/certs/ca-bundle.crt

# 使用RSA算法生成私钥
openssl genrsa -out k8s.io.key 2048

# 生成证书请求文件
openssl req -new -key k8s.io.key -out k8s.io.csr -subj '/CN=*.k8s.io/O=UCloud/ST=GuangDong/L=Shenzhen/C=CN'

# 依据上面生成的CA证书和CA私钥，根据证书请求文件生成证书
# -in 证书请求文件
# -CA ca的证书
# -CAkey ca的私钥
# -CAcreateserial
# -extensions 
openssl x509 -req -in k8s.io.csr -CA cacert.pem -CAkey cakey.pem -CAcreateserial -out k8s.io.crt -days 3650 -config ./openssl.cfg -extensions k8s.io
popd
```

