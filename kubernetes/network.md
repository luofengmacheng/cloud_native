## kubernetes中的网络

### 1 k8s中的网络设计原则及常见的网络插件

* 每个POD都有独立的IP地址，POD内所有容器共享同一个网络命名空间
* 所有POD都在一个直接连通的扁平网络中，可以通过IP直接访问

### 2 CNI网络插件

CNI是一个网络接口，它主要包含2个部分，一个是配置文件，一个是配置网络的二进制。另外，CNI不一定是给k8s用的，其他的程序也会使用。



### 3 Flannel(CoreOS)



### 4 Calico

### 5 Websocket(kubectl exec)

#### 5.1 什么是Websocket？它是为了解决什么问题？

要说什么是websocket，就需要先看下http的问题。http作为应用层的协议，通信的方式通常是单向一问一答：客户端发起请求，服务端响应。这种工作方式在大部分的WEB场景下也是够用的，但是，对于某些场景下的应用却有问题：

* 弹幕：需要实时将某个用户发送的弹幕转发给其他用户
* 代理：接收到请求后，需要实时转发给其他server

这些应用的特点是，服务器能够主动推送数据，客户端和服务器是全双工，这就是websocket的主要特点。

websocket就是为了解决服务端主动推送数据而产生的。

#### 5.2 协议分析

要了解一个协议，最简单的方式就是抓包看协议的工作方式：

![wireshark](https://github.com/luofengmacheng/cloud_native/blob/master/docs/network1.png)

图中总共抓了16个包，其中：

* 1->3：TCP建立连接
* 4->7：协议升级
* 8->13：数据交互
* 14->16：TCP断开连接

因此，websocket的连接建立和销毁过程跟http一样，不一样的就是在数据交互前，需要先对协议升级，然后再进行数据交互，并且数据交互时是直接基于TCP的，而不是在HTTP之上的。

第4个包是客户端向服务端发送协议升级请求，第5个包是服务端对客户端发送的第4个包的确认。

```
Connection: Upgrade\r\n
Sec-WebSocket-Key: cHgSMsfxKpnmpXM1KcHPOw==\r\n
Sec-WebSocket-Version: 13\r\n
Upgrade: websocket\r\n
```

其中比较重要的字段是Connection和Upgrade，表明需要将协议升级为websocket，Sec-WebSocket-Version是websocket的版本，现在一般都是13，Sec-WebSocket-Key是客户端用base64编码的随机字符串，用于进行websocket握手。

第6个包是服务端向客户端发送协议升级成功的通知，第7个包是客户端对服务端发送的第6个包的确认。

```
HTTP/1.1 101 Switching Protocols\r\n
Upgrade: websocket\r\n
Connection: Upgrade\r\n
Sec-WebSocket-Accept: eB70U2oE9e8g9MHF5BTL6gjhEGQ=\r\n
```

响应头表明协议升级成功，Connection和Upgrade字段跟客户端一样，Sec-WebSocket-Accept是用客户端发送的Sec-WebSocket-Key进行编码的字符串。

wireshark抓包的例子是个echo程序：客户端监听用户的输入，服务端收到数据后，再将数据发送给客户端，其中，用户输入了2次，每次有3个包。

第8个包是客户端发送给服务端的数据，websocket包含一个包头和实际的数据，包头中比较重要的字段有：Opcode，数据类型，这个直接用的文本，所以是Text，另外的Mask相关的不需要太多关注。

```
WebSocket
    1... .... = Fin: True
    .000 .... = Reserved: 0x0
    .... 0001 = Opcode: Text (1)
    1... .... = Mask: True
    .000 0011 = Payload length: 3
    Masking-Key: 4204cb9a
    Masked payload
    Payload
Line-based text data (1 lines)
```

第9个包是服务端发送给客户端的数据。第10个包是客户端对服务端发送的数据包的确认。

```
WebSocket
    1... .... = Fin: True
    .000 .... = Reserved: 0x0
    .... 0001 = Opcode: Text (1)
    0... .... = Mask: False
    .000 0011 = Payload length: 3
    Payload
Line-based text data (1 lines)
```

#### 5.3 gorilla/websocket for golang

golang中比较常用的websocket的就是gorilla的websocket，下面用一个echo的例子说明websocket的使用。

上面说过，websocket的连接建立和连接断开跟http一样，只是在数据交互之前需要先对协议升级。因此，客户端在连接服务端时需要告知要进行协议升级，而服务端在收到请求后，需要判断是否需要升级，如果需要升级，则进行协议升级，然后再进行数据交互。

``` golang
// echo server.go
import (
	"fmt"
	"net/http"

	"github.com/gorilla/websocket"
)

var upgrader = websocket.Upgrader{}

func echo(w http.ResponseWriter, r *http.Request) {
	if websocket.IsWebSocketUpgrade(r) {
		// 如果是websocket请求，则进行协议升级
		c, err := upgrader.Upgrade(w, r, nil)
		if err != nil {
			fmt.Errorf("upgrade: %v", err)
		}

		defer c.Close()

		// 读取数据，然后将数据发送给客户端
		for {
			mt, message, err := c.ReadMessage()
			if err != nil {
				fmt.Errorf("readmessage: %v", err)
				break
			}

			fmt.Printf("readmessage: %s\n", message)
			err = c.WriteMessage(mt, message)
			if err != nil {
				fmt.Errorf("writemessage: %v", err)
				break
			}
		}
	} else {
		// 如果不是websocket，直接发送一条提示
		w.Write([]byte("not websocket"))
	}
}

func main() {
	http.HandleFunc("/echo", echo)
	http.ListenAndServe("127.0.0.1:8080", nil)
}
```

``` golang
// echo client.go
import (
	"fmt"
	"log"
	"net/url"

	"github.com/gorilla/websocket"
)

func main() {
	// 指定协议为ws
	u := url.URL{
		Scheme: "ws",
		Host:   "127.0.0.1:8080",
		Path:   "/echo",
	}

	c, _, err := websocket.DefaultDialer.Dial(u.String(), nil)
	if err != nil {
		log.Fatal("dail: ", err)
	}

	defer c.Close()

	// 接收服务端发送来的数据，并打印输出
	go func() {

		for {
			_, message, err := c.ReadMessage()
			if err != nil {
				fmt.Errorf("read: %v\n", err)
				return
			}

			fmt.Printf("recv: %s\n", message)
		}
	}()

	for {
		// 读取用户输入端数据，发送给服务端
		data := ""
		fmt.Scanln(&data)

		// 发送数据时，数据类型为Text
		err := c.WriteMessage(websocket.TextMessage, []byte(data))
		if err != nil {
			fmt.Errorf("write: %v\n", err)
			return
		}
	}
}
```

#### 5.4 http的长连接和tcp的长连接

提到websocket时，经常会提到长连接，因为websocket可以让`连接保持`，然后服务端和客户端之间可以相互通信，而长连接也是让`连接不断开`，另外，tcp里面也有长连接到概念，它们分别有什么关系和区别呢？

##### 5.4.1 TCP

首先，长连接的目的是为了连接的复用，因为TCP在建立连接和断开连接时要经过三次握手和四次挥手，如果只是传输很少的数据，开销比较大，并且，每个连接只能使用一次，数据传输完毕就结束了的话，对于高并发的场景是不够的。

因此，TCP提供了长连接的机制，连接建立，数据交互完成后，也不立即断开，因为之后可能还需要传输数据，实现方式是进行探测。服务端会定期探测客户端的存活情况，完成对连接的管理。

在linux中，与TCP有关的配置有：

* net.ipv4.tcp_keepalive_time：长连接保持时间，默认值是7200秒，也就是2小时
* net.ipv4.tcp_keepalive_intvl：探测的时间间隔，单位是秒，默认值是75秒
* net.ipv4.tcp_keepalive_probes：探测的次数，默认值是9次

于是，TCP的长连接保活机制是：如果tcp_keepalive_time(默认是2小时)时间没有数据交互，就开启探测，每隔tcp_keepalive_intvl(默认是75秒)时间发送一次探测包，总共探测tcp_keepalive_probes(默认是9次)次，如果服务端一次响应都没有收到，则认为客户端已经断开连接，则服务端也断开连接，这样就完成了连接的管理。

另外，上面三个参数只是系统的配置，实际开发时如果要使用TCP的长连接，需要在`服务端`对套接字设置keepalive选项：

首先，需要先设置SO_KEEPALIVE选项用于启用TCP的长连接，然后再设置下面的几个参数。

* TCP_KEEPCNT：探测的最大次数，对应tcp_keepalive_probes
* TCP_KEEPIDLE：开始探测前连接的保持时间，对应tcp_keepalive_time
* TCP_KEEPINTVL：探测的时间间隔，对应tcp_keepalive_intvl

如果是C语言，可以直接调用setsockopt()设置上述的三个选项即可。对于golang，默认的net包不支持设置tcp的keepalive选项，如果要使用，就需要用其他的第三方包设置。

##### 5.4.2 HTTP

HTTP的长连接也是为了连接的复用。在http1.0中，长连接默认是关闭的，浏览器发送请求，服务端响应请求后，会自动断开连接，如果浏览器需要再次下载图片或者其他资源，需要重新建立连接。在http1.1中，长连接默认是打开的，浏览器发送请求，服务端响应请求后，不关闭连接，下次客户端请求图片或者其他资源，依然可以重用该连接。

因此，HTTP的长连接通常是服务器软件的行为。

以nginx为例，在nginx中与http keepalive相关的有三个配置：

* keepalive_requests：单个长连接最多处理的请求数，默认值是1000，也就是，当服务端处理完1000个请求后，自动断开连接
* keepalive_time：单个长连接最多持续的时间，默认值是1小时
* keepalive_timeout：最大的`空闲时间`，也就是如果这段时间没有数据交互，服务端就会自动断开连接

##### 5.4.3 Websocket

websocket是全双工协议，可以实现双向数据通信，由于websocket是基于TCP的，如果两端都不发送数据，默认情况下就会按照TCP来处理。

但是，即便使用TCP的keepalive也还是有问题：如果中间有代理，那么也无法保证端到端的长连接。因此，还是需要应用层的探测机制。

websocket提供了ping-pong的方式，其实就是服务端发送ping控制消息，客户端收到后返回pong控制消息，表明双方的连接关系是正常的，并且可以正常收发数据。

以golang为例，gorilla/websocket提供了SetPingHandler()和SetPongHandler()方法，分别用于设置ping和pong的回调函数，但是没有提供发送消息的方式，那就只能WriteControl()发送控制消息了。

如果TCP连接不断开，那么websocket在什么情况下会断开呢？是否会有超时时间呢？暂时还没有找到这块的资料。

##### 5.4.4 小结

长连接通常由`保活机制`实现，由`服务端`在`连接空闲一段时间`后发起探测，当多次探测都失败时，认为客户端出现问题，此时会断开连接，从而保证连接的可用性。

TCP是传输层的协议，它的长连接更多是为了在协议层面进行连接复用。而HTTP作为应用层协议，它的长连接更多的是让服务端在处理请求后不将连接断开，更偏向于业务上层。

### 6 网络虚拟化

* bridge：
* vxlan：
* macvlan：
* ipvlan：
* macvtap：

### 7 参考文档

* [Deep dive into docker overlay networks](https://blog.revolve.team/2017/04/25/deep-dive-into-docker-overlay-networks-part-1/)
* [深入浅出kubernetes中的CNI](https://github.com/helios741/myblog/blob/new/learn_go/src/2020/0303_k8s_cni/README.md)