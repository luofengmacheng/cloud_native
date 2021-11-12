## NFS部署

### 1 服务端

* yum install -y nfs-utils
* 创建共享目录：`mkdir /data/share`
* 编辑/etc/exports

```
# 将本地的/data/share共享给客户端，并且还带了一些选项用于控制读写和权限
/data/share $CLIENT_IP(rw,no_root_squash)
```
* systemctl start nfs-server

### 2 客户端

* yum install -y nfs-utils
* systemctl start nfs-mountd
* 展示服务端的可用挂载点：showmount -e $SERVER_IP
* 执行挂载：mount -t nfs $SERVER_IP:/data/share /data/remote