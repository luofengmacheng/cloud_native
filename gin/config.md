## 配置文件处理

每个程序都会使用到配置文件，现阶段最常用的配置文件格式是toml和yaml。通常，配置文件会进行分组，例如，通用配置可能包含程序的环境、程序监听的端口等，数据库配置可能包含数据库的IP、端口、名称、用户名和密码等。

### 1 toml配置文件解析

``` toml
[common]
listenip="0.0.0.0"
listenport=8080

[database]
host="127.0.0.1"
port=3306
dbname="test"
user="root"
passwd="root"
```

使用toml库解析上述toml配置文件，可以将配置文件解析为两个结构体：Com和Dt。

``` golang
package main

import (
	"fmt"
	"log"

	"github.com/BurntSushi/toml"
)

type Conf struct {
	Com *Common   `toml:"common"`
	Dt  *Database `toml:"database"`
}

type Common struct {
	ListenIp   string `toml:"listenip"`
	ListenPort int    `toml:"listenport"`
}

type Database struct {
	Host   string `toml:"host"`
	Port   int    `toml:"port"`
	Dbname string `toml:"dbname"`
	User   string `toml:"user"`
	Passwd string `toml:"passwd"`
}

func main() {
	var config Conf
	var path string = "./conf.toml"
	if _, err := toml.DecodeFile(path, &config); err != nil {
		log.Fatal(err)
	}

	fmt.Printf("%v %v\n", config.Com, config.Dt)
}
```

### 2 yaml配置文件解析

``` yaml
common:
  listenip: 0.0.0.0
  listenport: 80
database:
  host: 127.0.0.1
  port: 3306
  dbname: test
  user: root
  passwd: root
```

使用yaml库解析上述配置文件：

``` golang
package main

import (
	"fmt"
	"io/ioutil"

	"gopkg.in/yaml.v2"
)

type Conf struct {
	Com *Common   `yaml:"common"`
	Dt  *Database `yaml:"database"`
}

type Common struct {
	ListenIp   string `yaml:"listenip"`
	ListenPort int    `yaml:"listenport"`
}

type Database struct {
	Host   string `yaml:"host"`
	Port   int    `yaml:"port"`
	Dbname string `yaml:"dbname"`
	User   string `yaml:"user"`
	Passwd string `yaml:"passwd"`
}

func main() {
	yamlFile, err := ioutil.ReadFile("conf.yaml")
	if err != nil {
		fmt.Println(err.Error())
	}
	var config Conf
	err = yaml.Unmarshal(yamlFile, &config)
	if err != nil {
		fmt.Println(err.Error())
	}

	fmt.Printf("%v %v\n", config.Com, config.Dt)
}
```

### 3 通用解析

上面的解析方式有以下问题：

* 不同类型的配置文件需要使用不同的库进行解析(当然，解析的代码并不复杂)
* 未提供命令行参数的解析

使用`github.com/Akagi201/utilgo/conflag`可以同时解析json、toml和yaml，并且可以与命令行参数结合。

假设程序需要提供procs参数，可以在toml文件中设置：

``` toml
procs = 4
```

``` golang
package main

import (
	"flag"
	"fmt"
	"runtime"

	"github.com/Akagi201/utilgo/conflag"
)

type Conf struct {
	Procs int
}

var config Conf

func main() {
	flag.IntVar(&config.Procs, "procs", runtime.NumCPU(), "GOMAXPROCS")

	// set flags from configuration before parse command-line flags.
	args, err := conflag.ArgsFrom("conf.toml")
	if err != nil {
		panic(err)
	}
	flag.CommandLine.Parse(args)

	// parse command-line flags.
	flag.Parse()

	fmt.Println(config)
}
```

此时，配置就有三种给出的方式：

* 命令行
* 配置文件
* 默认值

以上的配置的优先级是从高到低的。