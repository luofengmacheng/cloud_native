## 日志处理

### 1 简介

日志是任何程序都逃不开的部分，日志既能帮助快速定位问题，也能帮助开发人员快速接手项目(因为日志中通常可以知道某些业务逻辑的调用流程)。golang中提供了基础的log库，另外还有比较常用的日志库logrus。

### 2 log

日志通常需要包含以下几个部分：

* 时间
* 打印日志的位置(文件名和行号)
* 日志级别
* 日志内容

为了对上述内容进行自定义，log中提供了以下几个常量让用户可以对格式进行自定义调整：

``` golang
const (
  // 日期: 2009/01/23
  Ldate = 1 << iota
  // 时间: 01:23:23
  Ltime
  // 毫秒级时间: 01:23:23.123123。该设置会覆盖 Ltime 标志
  Lmicroseconds
  // 完整路径的文件名和行号: /a/b/c/d.go:23
  Llongfile
  // 最终的文件名元素和行号: d.go:23
  // 覆盖 Llongfile
  Lshortfile
  // 标准日志记录器的初始值
  LstdFlags = Ldate | Ltime
)
```

因此，通常log都会带上三个flag：`Ldate|Lmicroseconds|Llongfile`。这里面少了级别字段，可以调用`log.SetPrefix()`设置日志开始的字符串，可以设置为日志级别。

整个log库包含以下几个部分：

1 Logger类的方法

* New()：第一个参数是io.Writer，可以用文件的对象；第二个参数是prefix，可以后续用SetPrefix()调整；第三个参数是flag，即上面的标志位
* formatHeader：根据用户提供的标志位写入日志开头的部分
* SetOutput/Writer：修改/获取日志的写入目的
* Output：写入日志的底层函数

* SetPrefix/Prefix：设置/获取日志的前缀
* SetFlags/Flags：设置/获取日志的flag，调整日志格式
* Print：直接打印
* Printf：带格式打印
* Println：打印并换行
* Fatal：Print+os.Exit
* Fatalf：Printf+os.Exit
* Fatalln：Println+os.Exit
* Panic：Print+panic
* Panicf：Printf+panic
* Panicln：Println+panic

2 log库的全局方法：

拥有Logger的除了formatHeader的所有方法。

log库默认会创建一个Logger对象：`var std = New(os.Stderr, "", LstdFlags)`，当直接调用log.Printf()时，其实是调用std对象的Printf()，然后Logger.Printf()会调用Logger.Output()写入日志，而Logger.Output()会调用Logger.formatHeader()写入日志开始的部分，然后写入用户打印的日志。

整个过程如下：

```
log.Printf() -> log.Logger.Printf() -> log.Logger.Output() -> log.Logger.formatHeader()
```

log库没有提供日志级别，也没有提供日志轮转，因此，如果要使用log库，建议用Prefix标识日志级别，然后分别用多个Logger对象分别打印不同级别的日志：

``` golang
package main

import (
  "log"
  "os"
)

func main() {

  fp, _ := os.OpenFile("run.log", os.O_CREATE|os.O_APPEND, 6)

  Info := log.New(fp, "INFO: ", log.Ldate|log.Lmicroseconds|log.Llongfile)
  Debug := log.New(fp, "DEBUG: ", log.Ldate|log.Lmicroseconds|log.Llongfile)
  Error := log.New(fp, "ERROR: ", log.Ldate|log.Lmicroseconds|log.Llongfile)

  Info.Printf("info data \n")
  Debug.Printf("debug data\n")
  Error.Printf("error data\n")
}
```

### 3 logrus

logrus有以下特性：

* 与标准的log库兼容
* 多种结构化的日志，便于后期处理，并与日志采集系统结合
* 通过Field机制对日志字段进行精细化的控制

#### 3.1 基本使用

logrus默认只会写入time、level、msg，需要调用SetReportCaller(true)才会写入func和file。

``` golang
package main

import (
  "os"
  "github.com/sirupsen/logrus"
)

var logger = logrus.New()

func main() {

  // 调用SetReportCaller(true)才会写入func和file
  logger.SetReportCaller(true)

  logger.WithFields(logrus.Fields{
    "animal": "walrus",
    "size":   10,
  }).Info("A group of walrus emerges from the ocean")
}
```

通常我们可能会将日志输出到某个特定的文件：

``` golang
package main

import (
  "os"
  "github.com/sirupsen/logrus"
)

var logger = logrus.New()

func main() {
   file, err := os.OpenFile("logrus.log", os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0666)
   if err == nil {
    logger.Out = file
   } else {
    logger.Info("Failed to log to file, using default stderr")
   }

  logger.WithFields(logrus.Fields{
    "animal": "walrus",
    "size":   10,
  }).Info("A group of walrus emerges from the ocean")
}
```

#### 3.2 日志级别

logrus中提供了7种日志级别，以下7种日志级别依次降低：

* PanicLevel：打印日志，然后调用panic
* FatalLevel：打印日志，然后调用Exit
* ErrorLevel：错误日志
* WarnLevel：警告日志
* InfoLevel：信息日志
* DebugLevel：调试日志
* TraceLevel：比调试更细粒度的信息

logrus默认的日志级别是InfoLevel，因此，默认情况下，只有调用InfoLevel及以上才会输出日志。

由于PanicLevel和FatalLevel会导致程序退出，它们的主要使用场景通常是程序启动时执行一些检查，如果检查不通过则退出。大部分情况下还是使用ErrorLevel和WarnLevel，ErrorLevel用于记录错误，WarnLevel用于记录警告，而InfoLevel用于记录一些关键的需要定位的信息。DebugLevel和TraceLevel由于日志过于详细，通常用于开发过程中问题的定位，也可以用于上线后日志级别的调整。

#### 3.3 日志轮转

logrus本身不提供日志轮转，需要依赖其他的库，例如file-rotatelogs。

``` golang
package main

import (
  "os"
  "github.com/sirupsen/logrus"
  rotatelogs "github.com/lestrrat-go/file-rotatelogs"
)

var logger = logrus.New()

func main() {
   
  file, err := rotatelogs.New(
    // 日志文件名，这里以年月日结尾
    "./logrus.log.%Y%m%d",
    // 链接名称，可以用于定位到最新的文件
    rotatelogs.WithLinkName("./logrus.log"),
    // 每隔24小时轮转一次(不一定是自然天)
    rotatelogs.WithRotationTime(24*time.Hour),
    // 最多保留30个文件
    rotatelogs.WithRotationCount(30),
  )
  if err == nil {
    logger.SetOutput(file)
  } else {
    logger.Info("Failed to log to file, using default stderr")
  }

  logger.WithFields(logrus.Fields{
    "animal": "walrus",
    "size":   10,
  }).Info("A group of walrus emerges from the ocean")
}
```
