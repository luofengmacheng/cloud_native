## CentOS 9使用Docker镜像实现交叉编译

### 1 安装Docker

``` shell
dnf config-manager --add-repo=http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
dnf makecache
dnf install -y docker-ce
systemctl start docker
```

之后就可以拉取镜像了，由于一些原因，无法从DockerHub拉取镜像，此时通常有两种办法，一种是找国内对应的镜像仓库，另一种方式是找代理。

经过测试发现，国内的镜像仓库是需要登录的，有些麻烦，好在，有些代理还是可以用的，例如：https://docker.1panel.live/，当需要拉取ubuntu时，可以使用：

``` shell
docker pull docker.1panel.live/library/ubuntu
```

### 2 安装QEMU模拟器

如果需要在x86环境构建arm平台代码，需要安装QEMU模拟机：

``` shell
docker run --rm --privileged docker.1panel.live/multiarch/qemu-user-static --reset -p yes
```

### 3 构建基础镜像

``` dockerfile
FROM arm64v8/ubuntu:16.04
RUN sed -i "s@http://.*archive.ubuntu.com@https://mirrors.tuna.tsinghua.edu.cn@g" /etc/apt/sources.list && \
    sed -i "s@http://.*security.ubuntu.com@https://mirrors.tuna.tsinghua.edu.cn@g" /etc/apt/sources.list && \
    apt-get update && \
    apt-get install -y gcc g++ cmake binutils-dev
CMD ["/bin/bash"]
```

与x86的主要区别是基础镜像要用`arm64v8/ubuntu`，之后就可以基于上述镜像去编译自己的程序。

### 4 参考文档

* [DockerHub镜像无法下载的多种解决方案](https://zhuanlan.zhihu.com/p/704119891)

