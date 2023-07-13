## Linux修改内核版本

### 1 目的

有时候需要对特定内核版本进行测试，因此，需要修改当前的内核版本。

### 2 安装内核包

首先执行`yum list kernel-*`查看当前是否有可用的对应版本的包，然后直接通过`yum install`安装对应的内核包即可。

如果当前yum源中没有，可以找其他有对应版本内核的源，例如，添加[ELRepo](http://elrepo.org/tiki/HomePage)或者[vault](https://vault.centos.org)仓库：`yum-config-manager --add-repo https://vault.centos.org/7.7.1908/os/x86_64`，使用yum-config-manager添加仓库时，要注意网站的路径下需要存在repodata/repomd.xml文件。

如果yum中也没有或者由于网络原因不能安装yum源，那可以单独去下载kernel包，由于某个发行版中可能没有更新kernel，因此，需要去找到该内核版本所在的发行版本，然后去对应的发行版本中找，例如要下载2.6.32-642.1.1.el6.x86_64版本的内核包，可以执行命令`wget https://vault.centos.org/6.8/updates/x86_64/Packages/kernel-2.6.32-642.1.1.el6.x86_64.rpm
`下载，然后使用rpm进行安装，安装过程中可能某些依赖也需要更新，可以找对应的yum源更新。

### 3 重启系统

如果系统安装了grub2，比如，/etc/default/grub文件存在，该文件中的GRUB_DEFAULT表示启动时默认选择哪个启动项，一般设置为saved，表示使用上次选择的系统。如果升级了内核版本，重启系统后，新安装的版本一般在最上面，因此，如果是升级内核，在安装新版本内核后，直接重启即可。

如果系统没有安装grub2，那只能先尝试重启，再登录时查看内核版本再确认。

### 4 相关资源

* CentOS历史版本的软件包：https://vault.centos.org/
* https://pkgs.org/
* http://elrepo.org/tiki/HomePage