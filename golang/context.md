## Context

### 1 context的使用场景

golang中最具特色的就是提供了语言级别的协程：goroutine。那么，协程之间也是需要进行同步的。golang提供了三种协程同步的机制：

* channel：协程之间通过通道进行数据同步，通常用于协程之间的数据交互
* waitgroup：通常用于等待若干协程执行结束再去完成其他工作
* context：通常用于协程的退出和超时机制

举个简单的例子，当协程A调用协程B，由于协程A调用完协程B后就去完成自己的工作，没有等待协程B结束，如果协程A结束了，协程B怎么知道呢？因此，就需要一种通知机制。

### 2 context的使用方法

上面说过，context的主要使用场景是正常退出和超时机制。

#### 2.1 父子协程正常退出

``` golang
import (
	"fmt"
	"time"
)

func DoSomeWork() {
	tick := time.NewTicker(time.Second)
	for {
		select {
		case <-tick.C:
			fmt.Printf("Do work...\n")
		}
	}
}

func main() {

	go DoSomeWork()

	time.Sleep(5 * time.Second)
}
```

上面的代码，通过time.NewTicker()创建定时器，然后使用for-select实现定时执行，使用goroutine执行这段逻辑，如果主函数(主goroutine)退出了，所有的子协程都会被干掉，因此，如果这段代码用在这种只执行一次就退出的场景没有问题，但是，如果用在服务端程序中，服务端创建大量这样的协程，而这样的协程是不会退出的，只有main()函数退出时，所有的子协程才会退出。也就是说，在服务端程序中，由于有大量的协程，而且协程之间可能会有很复杂的父子关系，子协程就需要一种机制可以知道父协程已经退出了，然后自己也退出协程。

``` golang
package main

import (
	"context"
	"fmt"
	"time"
)

func Child(ctx context.Context) {
	tick := time.NewTicker(time.Second)
	for {
		select {
		case <-ctx.Done():
			fmt.Printf("Parent exit, Child exit\n")
			return
		case <-tick.C:
			fmt.Printf("Do work...\n")
		}
	}
}

func Parent(ctx context.Context) {
	childCtx, cancelFunc := context.WithCancel(ctx)
	defer cancelFunc()

	go Child(childCtx)

	// do some work
	time.Sleep(3 * time.Second)
	fmt.Printf("Parent exit\n")
	
}

func main() {
	go Parent(context.Background())

	select {}
}
```

假设这是一个服务端程序，Parent协程的工作是调用Child()协程，然后做一些自己的工作(这里用睡眠3秒表示)，之后退出协程，Child()的工作就是定时执行。为了实现父子协程的通知，目的是让Parent()协程退出时Child()协程也会退出：

* 调用Parent()传入一个context.Background()的Context，这种Context通常作为根Context，或者说不知道应该传哪个Context，又或者没有现成的Context可以传递
* Parent()协程中context.WithCancel()函数根据传进来的父Context创建一个可以取消的Context，此时会返回一个新的Context和一个取消函数，将新的Context传递给Child()协程，然后在做完工作后，调用cancelFunc()通知子协程退出，其实就是往刚才传入的Context的done Channel中传入一条数据
* Child()协程定时打印输出，同时，也会监听Context.Done()返回的Channel，如果Channel中有数据，就认为父Context已经结束，此时就可以退出子协程。这里需要注意的是，这里退出协程不能使用break，只能使用return，使用break只能退出select，而不能退出for。

#### 2.2 超时机制

``` golang
package main

import (
	"context"
	"fmt"
	"time"
)

func Child(ctx context.Context) {
	tick := time.NewTicker(time.Second)
	for {
		select {
		case <-ctx.Done():
			fmt.Printf("Parent exit, Child exit\n")
			return
		case <-tick.C:
			time.Sleep(1 * time.Second)
			fmt.Printf("Do work...\n")
		}
	}
}

func Parent(ctx context.Context) {
	childCtx, cancelFunc := context.WithTimeout(ctx, 10*time.Second)
	defer cancelFunc()

	go Child(childCtx)

	// do some work
	time.Sleep(5 * time.Second)
	fmt.Printf("Parent exit\n")
}

func main() {
	go Parent(context.Background())

	select {}
}
```

对前面的代码只调整了一行，context.WithCancel()换成了context.WithTimeout()，也就是带有超时的Context，并且在创建Context时还带了超时时间(当前设置为10秒)。这样的效果就是，如果Child()执行时间超过超时时间，就会在超时时间到达时，自动调用取消函数，向done中发送取消信号；如果Child()执行时间未超过超时时间，那么就由Parent()调用。

### 3 Gin中的Shutdown

优雅退出的意思是不影响正在通信的连接，而只是不接收新的连接以及将空闲连接关闭。

``` golang
func (srv *Server) Shutdown(ctx context.Context) error {
	atomic.StoreInt32(&srv.inShutdown, 1)

	srv.mu.Lock()
	// 关闭所有的监听者，不接受新的连接请求
	lnerr := srv.closeListenersLocked()
	srv.closeDoneChanLocked()
	for _, f := range srv.onShutdown {
		go f()
	}
	srv.mu.Unlock()

	// 定时关闭空闲连接
	ticker := time.NewTicker(shutdownPollInterval)
	defer ticker.Stop()
	for {
		if srv.closeIdleConns() {
			return lnerr
		}
		select {
		case <-ctx.Done():
			return ctx.Err()
		case <-ticker.C:
		}
	}
}
```

Shutdown()用于优雅退出server，退出流程包括：

* 关闭所有监听者
* 关闭所有空闲的连接
* 等待连接变成空闲，然后关闭

Shutdown()函数的参数就是Context，因此，它的使用方法就是：

``` golang
svr := &httpServer{Addr: ":8080"}

ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()

if err := svr.Shutdown(ctx); err != nil {
	log.Errorf("svr.shutdown failed: %s\n", err)
}
```

给Shutdown()传递一个超时的Context，然后查看返回值，这里相当于保证了Shutdown()函数最多会执行3秒钟。