## 如何开发一个可以快速定位问题的程序

### 0 前言

在golang中，使用http或者gin的库可以快速开发一个web后端程序，但是，如果应用程序可能总会有bug，那么，如何开发出一个可以快速定位问题的程序呢？

对于程序来说，日志监控和进程监控是很重要的两个部分：

* 日志监控可以统计出成功率和失败率，然后可以依据成功率和失败率进行告警
* 进程监控可以保证进程存活，保证服务的可用

### 1 进程自监控

通常，实现进程监控有2种方式：

* crontab：直接在crontab里面添加定时任务，定期检查进程是否存活，然后进行启动
* 第三方进程：独立的agent进程负责监控，当检查进程不存活时，进行启动

上面的2种方式都依赖第三方的服务，第一个依赖crontab，但是，crontab也可能会有问题哦，第二个依赖第三方的agent程序，而且，这两种方式都会给系统的部署带来一定的复杂性。

因此，这里提出`进程自监控`的思路。

首先，进程会挂掉，通常是程序存在bug，但是，任何程序达到一定的复杂性，bug是避免不了的，因此，就需要一个足够简单的程序对主业务逻辑进行监控，当主业务逻辑挂掉后，进行自动重启。

moby提供了这样一个程序：[reexec](https://github.com/moby/moby/tree/master/pkg/reexec)

https://fankangbest.github.io/2017/11/26/Docker-reexec%E6%9C%BA%E5%88%B6%E5%88%86%E6%9E%90-v1-12-3/

https://www.reddit.com/r/golang/comments/bq9p08/difficulty_in_understanding_how_the_reexec/

``` golang
package main

import (
	"log"
	"os"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/moby/moby/pkg/reexec"
)

func init() {
	// 注册进程名和对应的入口函数
	reexec.Register("child_process", workProcess)
	if reexec.Init() {
		os.Exit(0)
	}
}

// 主业务逻辑，监听服务端口，处理请求
func workProcess() {
	gin.Default().Run("127.0.0.1:12345")
}

func main() {
	for {
		// 根据名称获取对应的命令对象
		cmd := reexec.Command("child_process")
		cmd.Stdin = os.Stdin
		cmd.Stdout = os.Stdout
		cmd.Stderr = os.Stderr

		var err error

		// 启动命令
		err = cmd.Start()
		if err != nil {
			log.Print("process start failed")
		} else {
			log.Print("process start success")
		}

		// 等待命令执行结束，这里也就是等待主业务逻辑推出
		// 1 当返回码为0时，不返回错误
		// 2 命令执行失败或者未执行结束，则返回ExitError，其他的错误一般都是I/O的问题
		err = cmd.Wait()
		if err != nil {
			log.Print("process wait failed")
		} else {
			log.Print("process wait success")
		}

		// 无论主业务逻辑是正常还是异常退出，都休息5秒，然后再次启动，当然，这里最好是推送一条消息，让运维能够感知到进程的重启，而不是默默的重启
		time.Sleep(5 * time.Second)
	}
}
```

### 2 日志跟踪

日志既可以用于定位问题，也能够用于采集，然后进行监控，但是，使用日志通常会遇到2个问题：

* 应该在哪里打印日志
* 日志的格式

对于第一个问题，建议是2块地方必须打印：模块接收请求和返回请求的地方，并且，返回请求的地方还可以计算耗时；调用其他模块和其他模块返回的地方。

对于第2个问题，通常来说，有2种方式，第一种是直接按照用户的字段进行打印，第2种方式是采用json打印，json打印的好处是方便日志采集模块解析，但是不方便人看。所以，为了方便日志的采集和解析，通常会进行自定义格式打印。另外，在定位问题时，通常需要查看一个请求从发起到结束经历了哪些模块，这时通常可以用trace进行跟踪。

在没有使用任何trace框架时，一个模块可以从请求中提取出trace，然后打印日志时带上该trace，定义好日志格式，那么日志采集模块就可以很好地进行解析和处理。

下面说明在golang的gin框架中的实现方式：

首先，在中间件模块中得到traceid，如果没有，则自行生成，并将traceid存储到context中，读取参数，并打印一条接收请求的日志，在执行实际的业务逻辑后，打印请求的整体返回以及耗时。

``` golang
func TraceMiddleware() gin.HandlerFunc {
	return func(c *gin.Context) {

		traceId := uuid.New().String()
		// 如果头部携带有请求的uid，则使用传送过来的uid，否则用自己生成的
		requestUuid := c.GetHeader("TRACEID")
		if requestUuid != "" {
			traceId = requestUuid
		}

		c.Set("trace", traceId)

		s, _ := ioutil.ReadAll(c.Request.Body)
		log.WithField("trace", traceId).WithField("parameter", string(s)).Info("receive request")
		c.Request.Body = ioutil.NopCloser(bytes.NewBuffer(s))

		t := time.Now()
		/*
		   c.Next()后就执行真实的路由函数，路由函数执行完成之后继续执行后续的代码
		*/

		c.Next()

		latency := time.Since(t).Seconds() * 1000
		status := c.Writer.Status()
		if status == http.StatusOK {
			log.WithFields(logrus.Fields{
				"trace": traceId,
				"status":       status,
				"latency":      latency,
				"client":       c.ClientIP(),
				"url":          c.Request.RequestURI,
			}).Info("success")
		} else {
			log.WithFields(logrus.Fields{
				"trace": traceId,
				"status":       status,
				"latency":      latency,
				"client":       c.ClientIP(),
				"url":          c.Request.RequestURI,
			}).Error("failed")
		}
	}
}
```

设置日志格式，这里的格式设置为：`时间|级别|trace|文件名:行数|用户关注的字段|message`。

``` golang
 var logger = logrus.New()

func LOG() *logrus.Logger {
	return logger
}

type LogFormatter struct {
}

func (f *LogFormatter) Format(entry *logrus.Entry) ([]byte, error) {
	line := fmt.Sprintf("%s|%s|%s|%s:%d|", entry.Time.Format("2006-01-02 15:04:05"), entry.Level, entry.Data["trace"], entry.Caller.File, entry.Caller.Line)

	data := []string{}

	for k, v := range entry.Data {
		if k != "trace" {
			data = append(data, fmt.Sprintf("%s=%v", k, v))
		}
	}

	return []byte(line + strings.Join(data, " ") + "|" + entry.Message + "\n"), nil
}

func init() {
	logger.SetReportCaller(true)

	logger.SetFormatter(new(LogFormatter))
}
```

然后，在实际的业务逻辑中，可以从context中获取trace，并写入日志，后续，在所有函数都加上context.Context参数，就可以直接获得trace信息。

``` golang
func MockHandler(c *gin.Context) {

	logger := logrus.LOG().WithField("trace", c.Value("trace").(string))
}
```