## kubernetes中的网络

### 1 CNI网络插件

### 2 k8s中的网络设计原则及常见的网络插件

* 每个POD都有独立的IP地址，POD内所有容器共享同一个网络命名空间
* 所有POD都在一个直接连通的扁平网络中，可以通过IP直接访问

### 3 Flannel

### 4 Calico

### 5 Websocket(kubectl exec)

