## vscode + golang 环境搭建

### 1 golang开发必备软件

* IDE: vscode
* 语言：go
* 插件及版本管理：git + github desktop

### 2 golang中的环境变量

* GOROOT golang语言自身的安装目录
* GOPATH 用户的工作目录，可以有多个目录，当有多个目录时，go get安装在第一个目录。在项目下有三个目录：src(存放源代码)、pkg(编译包时生成的静态链接库)、bin(代码编译后生成的可执行文件)。
* PATH 命令所在的目录，为了执行go命令，需要将$GOROOT/bin加入PATH，同时，为了可以执行个人项目中的命令，也可以将$GOPATH/bin加入PATH。

### 3 项目开发目录结构

所有的项目都放在$GOPATH指定的目录：每个项目都在src目录下有一个目录，每个项目还会包含多个包以及main.go。

```
go_project    // $GOPATH
    bin
        bin1    // 编译生成的可执行程序
        bin2
        bin3
    pkg
    src
        proj1    // 项目1
            app1    // 包1
            app2    // 包2
            main.go
        proj2    // 项目2
        proj3    // 项目3
```

针对每个项目，可以执行以下命令进行编译：

### 4 开发过程中的常用命令

* `go build` 编译，对普通包执行该命令会在pkg目录下生成静态库，针对main包执行该命令会在当前目录下生成可执行程序
* `go run` 执行，只能针对main包执行该操作
* `go install` 编译和安装，对普通包执行该命令会在pkg目录下生成静态库，针对main包执行该命令会编译生成可执行程序，并将可执行程序移到$GOPATH/bin目录下
* `go get` 从github或者其他平台上下载包，然后进行编译和安装，相当于`git clone + git install`