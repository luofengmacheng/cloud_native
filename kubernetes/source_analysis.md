## kubernetes源代码分析

### 1 关键目录以及对应的功能

| 目录 | 功能 |
| --- | ---- |
| build | 构建脚本 |
| cmd | 所有的二进制可执行文件的入口代码 |
| hack | 工具箱 |
| pkg |  |
| plugin | 插件，现在里面只有访问控制相关(admission和auth) |
| staging |  |
| test | 测试相关工具 |
| vendor | 依赖的第三方库 |

### 2 编译K8S

了解了kubernetes项目的大致目录结构，下一步就可以对kubernetes进行编译。golang比较好的一点就是依赖已经在仓库中，git clone后，只要make就行。

例如，如果要只编译kubectl就可以执行`make kubectl`。

这里面可能有变化的就是golang的版本，不同kubernetes编译依赖性的go版本不太一样，而且脚本中也会检查，如果编译时报的是版本相关的错，可以修改hack/lib/golang.sh中的kube::golang::verify_go_version()，在其中添加导出GOROOT和PATH的语句。

### 3 kubectl的调用路径

K8S将所有组件当作命令行工具实现，并且利用了cobra库

前面说过，cmd目录是所有可执行二进制的入口所在地，其中的每个目录都是一个main包，对应的就是一个组件，相当于就是一个命令，实现过程中使用了Cobra库。

该目录中的程序只做两件事：

* 创建Command：command := NewXXXCommand()
* 执行Command：command.Execute()

以kubectl为例：

``` golang
// cmd/kubectl/kubectl.go
command := cmd.NewDefaultKubectlCommand()

logs.InitLogs()
defer logs.FlushLogs()

if err := command.Execute(); err != nil {
	os.Exit(1)
}
```

因此，剩下要做的就是实现Command。

然后，每个组件的具体实现又对应一个单独的仓库，例如，对于kubectl来说，就是k8s.io/kubectl。

``` golang
// vendor/k8s.io/kubectl/pkg/
```

### 4 为K8S添加日志输出(kubectl)

为程序添加日志可以进行定位，并且了解具体执行流程，这里可以对kubectl添加日志。

k8s中普遍使用klog日志库，在程序中可以直接打印日志：

``` golang
klog.V(5).Infoln("print log")
```



### 5 以kube-apiserver为例说明

`cmd/kube-apiserver`目录是kube-apiserver组件的main包，其中的`apiserver.go`就是主程序，main函数干了三件事：

* `command := app.NewAPIServerCommand()` 组件好命令及对应的处理函数
* `logs.InitLogs()` 初始化日志
* `command.Execute()` 根据命令执行对应的函数

任何程序都需要处理命令行参数，因此，在`app.NewAPIServerCommand`里面的核心工作就是封装命令，里面主要用到的就是`github.com/spf13/cobra`库。

