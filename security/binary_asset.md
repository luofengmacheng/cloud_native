## 二进制漏洞检测方案

### 1 背景

镜像或者容器中，如果用户是通过包管理软件安装的程序，可以通过包管理软件获取对应的软件信息和版本信息，但是，如果用户自己编译了一个二进制，然后打包到镜像或者通过拷贝命令放到容器中，该如何识别该程序是否有风险呢？

漏洞检测方式有两种：

* 版本对比：将软件的版本跟漏洞库中影响的版本进行匹配，如果匹配上，说明有对应的漏洞，有一定的误报，而且需要先识别出软件名称和版本
* POC脚本执行：执行漏洞POC，查看是否有漏洞上报，准确率高，但是有些漏洞没有POC，而且这种方式可能对正在运行的系统造成影响

我们先尝试第一种方式，看能否获取二进制的版本信息(包含二进制使用的开源组件的版本信息)。

### 2 SCA(Software Composition Analysis)

SCA是一项对源代码或者二进制进行分析的技术，通过分析软件的一些组成部分和依赖等识别出软件可能包含的漏洞。例如，某个软件在开发过程中使用了一些公开的库，那么这些库可能包含一些漏洞。

#### 2.1 AquaSecurity

Aqua是一家容器安全初创公司，主要的开源产品有：[Trivy](https://github.com/aquasecurity/trivy)(容器漏洞扫描)、kube-bench(k8s的CIS安全基准测试)。而Aqua的容器安全平台使用的扫描引擎就是Trivy，而且可以设置选项[Scan standalone binaries in images](https://support.aquasec.com/support/solutions/articles/16000120202-configure-scanning-options)扫描不通过包管理安装的软件(使用开源的Trivy未发现可以扫描二进制，估计是在开源版本上通过plugin机制增加了额外的扫描能力)。

Trivy提供的命令有：

* trivy image：扫描镜像，通过分析操作系统的包管理器的配置文件得到安装的软件包和版本，然后通过软件包和版本得到对应的漏洞数据
* trivy fs：扫描本地文件系统的语言包和配置文件
* trivy rootfs：扫描rootfs，相当于扫描容器，扫描的方式跟镜像类似
* trivy repository：扫描远程的仓库
* trivy server：以server模式运行，然后提供接口
* trivy config：扫描配置文件中的密码和token等信息，例如：Dockerfile、k8s的yaml、cloudformation、Terraform
* trivy kubernetes：扫描k8s的漏洞

通过对trivy提供的命令以及使用的分析发现，开源的trivy采用的依然是以下三种方式：

* 通过操作系统包管理器(apt、yum)获取安装的软件信息，然后根据软件版本信息分析出漏洞
* 通过分析语言包的依赖库，以及依赖库的漏洞信息，得出整个软件的漏洞数据
* 扫描相关的配置文件(Dockerfile、yaml、Terraform)中的敏感信息(passwd、key、token)

#### 2.2 Synopsys Black Duck

[Black Duck](https://www.synopsys.com/zh-cn/software-integrity/security-testing/software-composition-analysis.html)提供以下几种能力：

* 程序分析：依赖分析(显然，也是通过分析配置文件)、码纹分析(可以识别C/C++开发的应用中使用的开源和第三方组件)、二进制分析、许可证分析
* 漏洞扫描

Black Duck包含detect和server，detect没有做太多的扫描动作，通常会将数据上传到server执行。

安装detect：`bash <(curl -s -L https://detect.synopsys.com/detect7.sh)`。

运行detect：`java -jar synopsys-detect-7.13.2.jar --detect.binary.scan.file.path=/usr/bin/ping --blackduck.offline.mode=true --detect.cleanup=false`。

但是detect在运行时一般需要制定server的地址，特别地，对于[二进制扫描模式](https://github.com/blackducksoftware/synopsys-detect/tree/master/src/main/java/com/synopsys/integration/detect/tool/binaryscanner)，需要将二进制上传到server进行扫描分析。而在实现原理上，则没有找到更多资料。

#### 2.3 腾讯

腾讯的云产品[T-Sec 二进制软件成分分析](https://cloud.tencent.com/product/bsca)能够对二进制提供自动化的分析能力，产品提供的功能包括：

* 开源组件识别
* 敏感信息扫描
* 漏洞扫描

腾讯云的二进制软件成分分析产品就是基于科恩的BinaryAI，而BinaryAI就是基于下面两篇论文：

* [基于图神经网络的二进制代码分析](https://zhuanlan.zhihu.com/p/96547586)：判断两个二进制是否是同一份源代码编译出来的。那么，在对待检测而进制而言，如果可以找到同一份源代码编译出来的其他二进制，那么他们应该就是从同一份源代码编译出来的，就可以得到待检测的二进制使用的软件和版本信息。当然，不同版本的软件可能修改的代码不是太多，可能造成版本识别不正确
* [基于跨模态检索的二进制代码-源代码匹配](https://zhuanlan.zhihu.com/p/271925850)：在给定二进制的情况下，匹配出对应的源代码。那么，在对待检测的二进制而言，如果可以找到对应的源代码，也就可以知道对应的软件和版本，即便是通过不同参数编译而成的。

在github上，[BinaryAI](https://github.com/binaryai)和[BinAbsInspector](https://github.com/KeenSecurityLab/BinAbsInspector)则都不太活跃。

腾讯的方案，利用AI模型对开源软件的数据进行训练，然后对待检测的二进制进行匹配，找到相似的二进制或者对应的源代码，基本可以确定使用的软件和版本。基于AI模型的方法比较重要的是数据集，BinaryAI每天会自动跟踪开源平台上的项目变化，对数据集进行更新。目前，BinaryAI已经采集全网主流C/C++项目的数万个仓库的上百万个版本，累计百亿C/C++源代码特征文件。

#### 2.4 悬镜安全

悬镜安全是一家做软件供应链安全的公司，它开源了[OpenSCA](https://github.com/XmirrorSecurity/OpenSCA-cli)，根据它所支持的检测能力看，它的实现方式是基于对包管理软件的配置文件的分析。

例如，对于golang，通过分析go.mod和go.sum文件，得出该软件依赖的包和版本，可以看出，这种方案其实还是源代码级别的分析。

#### 2.5 小结

基于对上述产品的调研，相对比较公开的SCA实现方式通常是：

* 分析操作系统的包管理软件
* 分析语言包的配置文件

而其他的实现方式，例如大数据或者AI，要么比较封闭，查不到太多资料，要么需要利用AI+大数据+算法实现，比较复杂。

### 3 Sandbox

容器是基于namespace、cgroup和chroot技术构建成的运行单元，但是多个容器之间依然是共享内核的，容器还是可能逃逸到宿主机，会对其他容器造成安全隐患，因此，通常有两种方式能够提供比原生容器更高的隔离性：

* 基于虚拟机，例如KVM或者Xen，在虚拟机之上搭建用户内核，各虚拟机之间完全隔离，但是这种方式的缺点在于，启动时间较长、消耗的资源更多
* 基于规则，例如Seccomp、SELinux和AppArmor，通过限定用户程序能够执行的操作达到隔离，缺点在于，规则比较复杂，而且内核依然是共用的

Sandbox是一种更加安全的环境，比虚拟机轻量的同时比容器更加安全，可以在该环境中执行POC测试，从而检测二进制的漏洞。

#### 3.1 gVisor(Google)

gVisor通过伪装成内核，对用户程序进行响应，由用户态进程执行系统调用。

gVisor有两个组件：

* Sentry：用户态的应用程序内核，本身运行在用户态，而且使用Seccomp限制了操作，当应用程序执行系统调用时，Sentry会收到请求，它自身会处理一些逻辑，无法执行的操作会通过9P协议转发给Gofer。
* Gofer：Gofer是个运行在宿主机的进程，当Gofer收到请求后，会处理所有对资源的访问，当然，也会提供额外的隔离。

于是，当容器中的进程执行系统调用时，需要有种机制截获该系统调用，然后转发给Sentry，Sentry在执行时，如果需要IO操作，则转发给Gofer。根据截获系统调用的机制区分，有两种方式：KVM和ptrace。

gVisor的安装以及作为docker的运行时：[gVisor Installation](https://gvisor.dev/docs/user_guide/install/)

一些重要的选项：

* --platform=kvm/ptrace：默认使用ptrace，使用kvm时，宿主机需要加载kvm模块
* --overlay：默认情况下，在容器中读写文件，操作的还是宿主机上面的文件，如果想要完全隔离，可以设置该参数
* --network=sandbox/host/none：默认使用sandbox网络，容器和宿主机可以通信，并做转发；如果设置为host时，则使用宿主机的网络；如果设置为none时，容器中则只有loopback，容器不能与宿主机通信
* --debug：启用debug
* --debug-log=/tmp/runsc/：debug日志路径

gVisor的一些问题：

* 本身对内核版本要求高(使用了user namespace，3.8开始引入，3.10需要修改配置/proc/sys/user/max_user_namespaces)
* Sentry没有实现所有的系统调用，部分程序可能无法运行
* 没有在大规模场景下使用过，可能有很多潜在的bug

#### 3.2 kata-containers

[kata-containers](https://github.com/kata-containers/kata-containers)是一个基于轻量虚拟机实现的容器化技术，它有容器的性能，但是提供比容器更好的隔离性和安全性，可以将它理解为一个经过裁剪的虚拟机。

分别下载containerd和kata：

``` shell
tar -C / -xf cri-containerd-cni-1.5.2-linux-amd64.tar.gz

tar -C / -xf kata-static-2.5.1-x86_64.tar.xz

cp -f containerd.service /etc/systemd/system/containerd.service 

mkdir /etc/containerd
cp -f config.toml /etc/containerd/config.toml

systemctl restart containerd

ctr image pull docker.io/library/busybox:latest
```

然后运行一个容器：`ctr run --runtime "io.containerd.kata.v2" --rm -t docker.io/library/busybox:latest test-kata uname -r`，会发现容器的内核跟宿主机的内核是不一样的，说明它不是传统的容器方式运行。

### 4 总结

根据以上的调研，如果使用版本对比的方式得到二进制漏洞，有以下方式：

* 基于包管理的软件清点，包括操作系统(yum、apt)和应用软件(pip等)
* 采集DockerHub中的主流软件的主程序的哈希值，形成知识库，然后用哈希值对比的方式获取软件信息
* 利用gVisor在沙箱中运行容器中的主程序的命令，得到应用的版本号，可以优化现有的容器进程的软件版本清点的卡死、影响业务的问题
* 基于腾讯开源的BinaryAI库或者BinAbsInspector建立自己的知识库(成本较高)

如果使用POC执行的方式得到二进制漏洞，只能基于一些不会对系统造成影响的环境进行测试，例如，gvisor和kata-containers。
