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

这里面可能有变化的就是golang的版本，不同kubernetes编译依赖性的go版本不太一样，而且脚本中也会检查，如果编译时报的是版本相关的错，可以修改hack/lib/golang.sh中的`kube::golang::verify_go_version()`，在其中添加导出GOROOT和PATH的语句。

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
// vendor/k8s.io/kubectl/pkg/cmd/cmd.go
func NewDefaultKubectlCommand() *cobra.Command {
	return NewDefaultKubectlCommandWithArgs(NewDefaultPluginHandler(plugin.ValidPluginFilenamePrefixes), os.Args, os.Stdin, os.Stdout, os.Stderr)
}

// NewDefaultKubectlCommandWithArgs creates the `kubectl` command with arguments
func NewDefaultKubectlCommandWithArgs(pluginHandler PluginHandler, args []string, in io.Reader, out, errout io.Writer) *cobra.Command {
	cmd := NewKubectlCommand(in, out, errout)

	if pluginHandler == nil {
		return cmd
	}

	if len(args) > 1 {
		cmdPathPieces := args[1:]

		// only look for suitable extension executables if
		// the specified command does not already exist
		if _, _, err := cmd.Find(cmdPathPieces); err != nil {
			if err := HandlePluginCommand(pluginHandler, cmdPathPieces); err != nil {
				fmt.Fprintf(errout, "Error: %v\n", err)
				os.Exit(1)
			}
		}
	}

	return cmd
}

func HandlePluginCommand(pluginHandler PluginHandler, cmdArgs []string) error {
	// 将kubectl后面所有的非选项的参数保存起来，直到找到一个选项
	// kubectl test abc -v=7
	// 那么，就将test和abc放到数组中
	var remainingArgs []string
	for _, arg := range cmdArgs {
		if strings.HasPrefix(arg, "-") {
			break
		}
		remainingArgs = append(remainingArgs, strings.Replace(arg, "-", "_", -1))
	}

	if len(remainingArgs) == 0 {
		// the length of cmdArgs is at least 1
		return fmt.Errorf("flags cannot be placed before plugin name: %s", cmdArgs[0])
	}

	foundBinaryPath := ""

	// 通过上面的保存的参数查找程序
	// 例如，kubectl test abc
	// 先根据test-abc查找，如果找不到，就查找abc
	// 具体的查找逻辑则交给插件处理器完成
	// 默认的插件处理器会调用exec.LookPath()在PATH中查找，另外，查找时会加上kube前缀
	for len(remainingArgs) > 0 {
		path, found := pluginHandler.Lookup(strings.Join(remainingArgs, "-"))
		if !found {
			remainingArgs = remainingArgs[:len(remainingArgs)-1]
			continue
		}

		foundBinaryPath = path
		break
	}

	if len(foundBinaryPath) == 0 {
		return nil
	}

	// 如果找到了插件程序，就调用插件处理器的Execute()执行，并将剩余的参数以及环境变量传给程序
	if err := pluginHandler.Execute(foundBinaryPath, cmdArgs[len(remainingArgs):], os.Environ()); err != nil {
		return err
	}

	return nil
}
```

其中，`NewKubectlCommand()`函数就是将kubectl所有的子命令构成成一个整体的命令，然后查找下是否有该命令，如果没有就当作插件处理，而`HandlePluginCommand()`就是处理kubectl插件的逻辑。

因此，就知道如何开发kubectl插件：

* 创建一个以`kubectl-`开头的程序(无论该程序用什么语言开发)，例如，`kubectl-github`
* 给程序添加执行权限
* 将该程序放到PATH的任何一个路径中(当然，最好是便于管理的目录)

然后，就可以执行`kubectl plugin list`查看到刚才创建的程序，执行`kubectl github`就可以执行刚才的插件程序，并且可以将剩余的参数都传给插件程序。

对于某个子命令而言，就位于`vendor/k8s.io/kubectl/pkg/cmd/`中的某个目录，例如，get子命令就位于`vendor/k8s.io/kubectl/pkg/cmd/get/get.go`。

总的来说，当执行一个命令时，例如，`kubectl get pods`，客户端只需要做三件事：

* 加载配置文件
* 向服务端请求数据
* 对返回的数据进行格式化输出

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

### 6 其他相关仓库

#### 6.1 开发环境

* minikube：在本地启动k8s，通常用于开发和测试

#### 6.2 开发和对应的实例

* cli-runtime：客户端工具库
* sample-cli-plugin：kubectl的plugin的简单示例

* client-go：k8s的客户端SDK
* sample-controller：controller的简单示例

* apiserver：创建kube-apiserver的库
* sample-apiserver：使用apiserver库的简单示例
* kube-aggregator：

* cloud-provider：cloud-provider的接口
* cloud-provider-sample：cloud-provider的简单示例

* cri-api：CRI的接口，可以用该库对接不同的运行时
* csi-api：CSI的接口

#### 6.3 开发工具库

* apimachinery：
* code-generator：
* ingress-nginx：NGINX的ingress控制器
* kube-openapi：
* klog：日志库

#### 6.3 运维

* kubeadm：k8s的安装、升级和管理
* kops：在云上对k8s进行安装、升级和管理
* cri-tools：基于CRI库实现的管理工具，功能类似docker客户端

#### 6.4 监控

* kube-state-metrics：获取集群级别的指标
* node-problem-detector：检测Node的问题
* metrics：
