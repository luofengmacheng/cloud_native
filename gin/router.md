## 路由管理

### 1 简介

web程序最重要的部分就是路由，路由可以告诉客户端，当调用某个url时，就会去执行某个业务逻辑，该业务逻辑执行结束后，就会将结果返回，于是前端就可以拿到后端的数据。golang有默认的http库，开发可以直接在golang中启动webserver，不需要像python或者php需要依赖其他组件。

* net/http：golang标准库，其中包含了基础的http客户端和服务端的实现
* httprouter：一个高性能的http路由库
* gin：基于httprouter实现的web库
* gollira/mux

### 2 net/http

``` go
package main

import (
	"fmt"
	"net/http"
)

func Hello(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "World")
}

func main() {
	http.HandleFunc("/", Hello)
	http.ListenAndServe(":8080", nil)
}
```

以上是个最简单的web程序，用户可以调用`http://localhost:8080/`就可以返回`World`。上面的程序里面最重要的是两个部分：

* http.HandleFunc()
* http.ListenAndServe()

下面依次介绍上面的两个部分。

``` go
var DefaultServeMux = &defaultServeMux

var defaultServeMux ServeMux

func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	DefaultServeMux.HandleFunc(pattern, handler)
}

type ServeMux struct {
	mu    sync.RWMutex
	m     map[string]muxEntry
	es    []muxEntry // slice of entries sorted from longest to shortest.
	hosts bool       // whether any patterns contain hostnames
}

type HandlerFunc func(ResponseWriter, *Request)

func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	if handler == nil {
		panic("http: nil handler")
	}
	mux.Handle(pattern, HandlerFunc(handler))
}

func (mux *ServeMux) Handle(pattern string, handler Handler) {
	mux.mu.Lock()
	defer mux.mu.Unlock()

	if pattern == "" {
		panic("http: invalid pattern")
	}
	if handler == nil {
		panic("http: nil handler")
	}
	if _, exist := mux.m[pattern]; exist {
		panic("http: multiple registrations for " + pattern)
	}

	if mux.m == nil {
		mux.m = make(map[string]muxEntry)
	}
	e := muxEntry{h: handler, pattern: pattern}
	mux.m[pattern] = e
	if pattern[len(pattern)-1] == '/' {
		mux.es = appendSorted(mux.es, e)
	}

	if pattern[0] != '/' {
		mux.hosts = true
	}
}
```

当调用`http.HandleFunc()`时，会调用默认的DefaultServeMux的HandleFunc()函数，DefaultServeMux是默认的ServeMux类型的对象defaultServeMux的指针，而HandleFunc()会调用ServerMux的Handle()方法。

Handle()方法有两个参数，第一个是路由pattern，第二个参数handler是一个函数类型：`type HandlerFunc func(ResponseWriter, *Request)`，具体逻辑就是将pattern和handler保存到一个map中。

``` go
func ListenAndServe(addr string, handler Handler) error {
	server := &Server{Addr: addr, Handler: handler}
	return server.ListenAndServe()
}
```

### 3 httprouter

### 4 gin中的路由