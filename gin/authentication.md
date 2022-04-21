## 登录鉴权(Cookie、Session、Jwt、CAS、SSO)

### 1 前言

登录鉴权是任何一个网站都无法绕开的部分，当系统要正式上线前都会要求接入统一登陆系统，一方面能够让网站只允许合法的用户访问，另一方面，当用户在网站上进行操作时也需要识别操作的用户，用作后期的操作审计。

### 2 Cookie & Session

由于HTTP是`无状态`的，也就是说，访问某个网站后，下一次再次访问，网站不能够识别出两次访问是否是同一个客户端。因此，为了能够对用户做验证，保障网站只能被合法的用户访问，需要将HTTP改成有状态的，此时就需要Cookie和Session配合使用。

需要识别当前HTTP请求是谁发起，就需要在后端存储用户信息，例如，用户ID、用户名、用户部门等，这些信息就存储在Session中，但是，在一段时间，可能有大量用户访问网站，也就是说，后端应该是存储一个字典，那么，哪一个用户才是当前这个请求的呢？这时就需要使用Cookie，Cookie是存储在客户端(浏览器)中的信息，将Session字典的键存储到客户端，当用户访问网站时，就会携带该Cookie，后端收到请求后，根据Cookie中的键就可以找到Session字典对应的值，也就能够识别当前用户的信息。

![](https://github.com/luofengmacheng/cloud_native/blob/master/gin/pics/session_cookie.png)

如上图所示，Cookie的存储形式也相当于KV，因此，最重要的属性就是名字和值。当前浏览器存储了SESSIONID=23456的Cookie，当用户通过该浏览器访问后端服务时，就会携带该Cookie(至于为什么会携带，这是Cookie本身的机制，当然还要看Cookie所属域名)，后端服务收到请求后，读取其中的Cookie，然后查询Session，就可以知道当前访问的用户是Hanmeimei。

上述过程只涉及到用户已经登陆，并且通过Session和Cookie关联用户信息，另外一部分比较重要的就是用户登陆，例如，如果后端收到请求后，从后端Session中查询不到用户信息该怎么办呢？

### 3 SSO & Jwt & CAS

在登陆部分，经常听到或者看到的有三个单词：

* SSO(Single Sign On)：单点登录，当用户登录一个平台后，再去访问关联的其他平台，不需要重新登录，例如，如果登录了OA系统，再登录内部的gitlab、云平台等都可以直接访问。所以，SSO描述的更多的是一种现象，也就是说，符合上述的现象都可以称之为SSO。
* JWT(Json Web Token)：基于Token的WEB认证方式，与上面使用Session存储用户信息的方式不同，这种方式在登陆后会将用户信息经过加密成Token，前端收到后存储到Cookie或者LocalStorage，当下一次访问时，将Token放到请求头的Authorization。所以，JWT指的更多的是用户信息的存放方式，这种方式在发起后端API请求时也经常使用：先调用/api/login接口，返回Token，得到Token后，直接访问其他业务接口。
* CAS(Central Authentication Service)：统一身份认证服务，是SSO的一种实现，在实现过程中，也集成了JWT，将用户信息放到Token中。所以，严格意义上来说，CAS是一套用于实现SSO的系统，或者说，CAS是一个认证服务器，提供用户登陆验证的能力。

根据上面所说的，CAS是一套认证系统，它可以用于生成用户登陆的Token，而客户端可以以JWT的方式进行使用，例如拿到了之后存放到Cookie中。

### 4 gin中使用SSO

完整的CAS登陆流程在网上有很多，这里也不再赘述。下面就来说说，在gin中如何使用SSO接入公司的单点登录。

系统需要接入单点登录，就需要引入web的中间件：中间件是web框架提供的一种能力，可以在web的请求处理流程中加入业务自己的部分，例如，登陆验证、日志记录、时耗记录等。

具体流程如下：

* 在中间件中判断用户是否登录，如果没有登录，则跳转到CAS的URL，CAS会判断用户是否登录，如果没有登录会跳转到登录页面
* 当用户在CAS的登录页面输入用户名和密码登录后，会跳转到业务系统，并且会携带ticket，业务系统根据ticket去CAS验证，如果CAS验证通过，会返回用户信息给业务系统
* 业务系统将用户信息保存到Session中，后续的请求就通过Session得到用户信息

```golang
// main.go 为gin添加中间件
r := gin.Default()
r.Use(Authenticate())

// middleware.go
func Authenticate() gin.HandlerFunc {
	return func(c *gin.Context) {

		// 从session中获取用户信息
		session := sessions.Default(c)
		userName := session.Get("user_name")
 
		if userName == nil || userName == "" {

			// 用户在当前平台没有登录，则需要获取ticket
			// 这里的逻辑在不同的实现中可能会不一样，
			// 有的可以直接从query参数获取ticket，有的是直接放在cookie中
			ticket, _ := c.Cookie("SESSIONID")
			if ticket == "" {
				// 没有用户的cookie，说明没有通过登录验证，需要把session中的用户置空
				session.Set("user_name", "")
				session.Set("user_id", "")
				c.Next()
				return
			}

			// 通过ticket调用CAS接口获取用户信息，
			// 如果未获取到，说明用户确实没有登录，则跳转到CAS登录页面
			// 在跳转时需要带上当前的页面地址用于登录成功后的跳转
			user := handler.GetLoginUser(ticket)
			if user.UserId == "" {
				params := url.Values{}
				params.Add("service", fmt.Sprintf("http://%s", c.Request.URL.String())
 
				c.Redirect(http.StatusFound, fmt.Sprintf("%s?%s", config.Config.AuthLoginUrl, params.Encode()))
				return
			} else {

				// 获取到用户信息，则将用户信息写入session
				session.Set("user_name", user.UserName)
				session.Set("user_id", user.UserId)
				session.Save()
 
				c.Next()
			}
		} else {
 
			c.Next()
		}
	}
}
```

### 5 前后端分离

现在的应用架构都采用前后端分离的方式，那么前后端分离的SSO跟非前后端分离有什么区别呢？

比较大的区别应该是在跳转部分的service参数：如果是前后端不分离，那么后端接收到的路由就是实际的页面地址，因此，如果登录成功跳转回来的话是可以直接用后端请求到的地址。如果前后端分离，跳转回来的页面肯定应该是前端的地址，这时候，跳转动作就只能在前端进行：

```javascript
    // 获取用户信息
    async getUserName() {
      let token = getCookie("SESSIONID")
      if (token === "") {
        window.location.href = 'https://cas.example.com/cas/login?service=' + window.location.href
      } else {
        const out = await getUserProfile()
        
        this.username = out.Data
        
        if (this.username === '') {
          window.location.href = 'https://cas.example.com/cas/login?service=' + window.location.href
        }
      }
    }
```

另外，在前后端分离的情况下，如果前端和后端不在同一个nginx下，会出现跨域问题。

### 6 跨域

跨域问题出现的原因是浏览器的同源访问限制策略，同源指的是相同的协议(例如，都是http)、域名、端口。为了安全起见，当浏览器发起XHR请求时，浏览器会查看当前源和请求的源是否相同，如果不相同就会禁止访问。

因此，跨域问题的出现有两个要素：

* 不同源
* XHR请求

如何解决跨域的问题呢？

常见的方案有三种：

* 修改浏览器设置，禁用浏览器安全策略
* JSONP，利用了非XHR请求不存在跨域，但是它只支持HTTP的GET请求
* CORS，在响应头增加Access-Control-Allow-Origin选项，说明服务端允许访问的域名
* 修改nginx或者apache的配置，当然也是在响应头增加Access-Control-Allow-Origin选项