## Golang模块化开发

### 1 前言

进行Golang开发时，通常会用到模块，例如github.com上的模块，或者自己开发的本地模块。

### 2 goproxy

当项目中用到了如github.com、golang.org下面的模块时，就需要从对应的网站下载包，由于墙的原因，这通常不是一件轻松的事情。

此时可以使用goproxy，设置GOPROXY环境变量，那么在下载包时就会从代理上下载。

例如：在/etc/profile中设置GOPROXY，`export GOPROXY=https://goproxy.io`，也可以使用阿里云的代理仓库：`https://mirrors.aliyun.com/goproxy/`。

此时就可以用`go get -u`对包进行更新。

### 3 Go Module

Golang 1.12正式支持Module，同时也支持`go mod`命令。

在没有Go Module的时代，如果要使用其它的模块，就需要用`go get`安装对应的模块，然后在代码中引用，或者使用第三方的包管理工具。有了官方提供的Go Module，可以自动进行所需模块的安装。

使用方法：

* 在项目根目录执行模块初始化命令：`go mod init`，会生成go.mod文件
* 编写业务逻辑，在程序中可以引用其它的模块，并将模块存储到`$GOPATH/pkg/mod`下
* 项目根目录下运行`go build`进行编译，会自动下载所需的模块，生成go.sum，并将程序中引入的外部包放到go.mod中，同时也会生成最终的可执行程序
