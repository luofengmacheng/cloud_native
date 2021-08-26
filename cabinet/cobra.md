## 命令行工具构建：Cobra

### 1 前言

日常工作中经常会见到或者使用到大量命令行工具，例如git、go、docker等，如何快速构建这样的命令行工具呢？当然，也可以自己实现，但是比较常用的还是基于Cobra库实现。

### 2 快速入门

基于Cobra库的程序的常用结构如下：

```
appName/
    cmd/
        add.go
        your.go
        commands.go
        here.go
    main.go
```

cmd目录中是具体的某个命令的实现逻辑，而main.go中的代码非常简单：

``` golang
package main

import (
  "{pathToYourApp}/cmd"
)

func main() {
  cmd.Execute()
}
```

为了能够快速生成上述的项目结构，可以使用Cobra提供的命令行工具cobra自动生成：

```
go get github.com/spf13/cobra/cobra
mkdir -p newApp && cd newApp
cobra init --pkg-name github.com/spf13/newApp
```

这样就会在newApp目录下生成一个main.go文件以及cmd目录，而cmd目录下就有一个root.go文件。

以docker命令为例，有以下常用命令：

```
docker images
docker image ls
```

可以直接使用cobra命令帮助生成命令的逻辑：

```
cobra add images
cobra add image
cobra add ls -p 'imageCmd'
```

有了上面的命令的框架，只需要修改cmd里面的cobra.Command中的参数对应的值，然后实现Run()方法就行。

既然是命令行工具，那就必然要涉及到配置和选项的解析，需要调整cmd/root.go中的两个地方：

* 在init()中添加选项声明
* 如果需要将配置读取到全局变量，则需要在initConfig()后面增加一个Unmarshal的逻辑

``` golang
// root.go的初始化函数
func init() {
	// 调用配置初始化函数，当任何一个命令执行时都会执行该函数
	cobra.OnInitialize(initConfig)

	// 这里可以声明命令行选项和配置
	// cobra提供了两种选项形式：
	// 持久选项：在当前命令以及子命令支持的选项
	// 局部选项：只在当前命令支持的选项
	rootCmd.PersistentFlags().StringVar(&cfgFile, "config", "", "config file (default is $HOME/.docker_cmd.yaml)")

	rootCmd.Flags().BoolP("toggle", "t", false, "Help message for toggle")
}

// 从配置文件或者环境变量读取配置
func initConfig() {

	// 使用viper设置配置文件路径
	if cfgFile != "" {
		// Use config file from the flag.
		viper.SetConfigFile(cfgFile)
	} else {
		// Find home directory.
		home, err := homedir.Dir()
		if err != nil {
			fmt.Println(err)
			os.Exit(1)
		}

		// Search config in home directory with name ".docker_cmd" (without extension).
		viper.AddConfigPath(home)
		viper.SetConfigName(".docker_cmd")
	}

	// 从环境变量读取配置
	viper.AutomaticEnv()

	// 从配置文件读取配置
	if err := viper.ReadInConfig(); err == nil {
		fmt.Println("Using config file:", viper.ConfigFileUsed())
	}
}

```

### 3 模拟docker命令行

下面就用Cobra来模拟docker的命令行。

```
mkdir -p docker && cd docker
cobra init --pkg-name github.com/luofengmacheng/docker
go mod init github.com/luofengmacheng/docker
cobra add images
cobra add image
cobra add ls -p 'imageCmd'

go build
./docker images
./docker image ls
```

如果只是模拟效果，不实现具体逻辑的话，完全不需要自行编码。