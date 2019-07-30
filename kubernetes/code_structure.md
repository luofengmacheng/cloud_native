## kubernetes代码结构

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

### 2 以kube-apiserver为例说明

`cmd/kube-apiserver`目录是kube-apiserver组件的main包，其中的`apiserver.go`就是主程序，main函数干了三件事：

* `command := app.NewAPIServerCommand()` 组件好命令及对应的处理函数
* `logs.InitLogs()` 初始化日志
* `command.Execute()` 根据命令执行对应的函数

任何程序都需要处理命令行参数，因此，在`app.NewAPIServerCommand`里面的核心工作就是封装命令，里面主要用到的就是`github.com/spf13/cobra`库。

