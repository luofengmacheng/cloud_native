## lua使用libcurl获取数据并解析json的问题

### 1 使用libcurl获取数据

lua中可以使用libcurl发起http请求：

``` lua
c = curl.easy_init() 
c:setopt(curl.OPT_SSL_VERIFYHOST, 0); 
c:setopt(curl.OPT_SSL_VERIFYPEER, 0); 
c:setopt(curl.OPT_URL, "https://"..ip.."/") 
c:setopt(curl.OPT_POSTFIELDS, "Username=admin&Password=admin&Frm_Logintoken=&action=login")

c:setopt(curl.OPT_WRITEFUNCTION, function(buffer)
　　　　return #buffer
　　end)

c:perform()
```

以上就是使用curl的基本用法，先通过easy_init()进行初始化，然后通过setopt()设置请求的参数和选项，最后通过perform()执行操作并得到返回数据和状态码。

当通过http调用k8s的接口时，常规的接口没有太大区别，但是，为了实时获取数据的变化通知，k8s提供了watch机制，可以通过watch机制获取数据的变更。

如果想使用watch机制调用k8s的http接口，只需要给url加上watch=true参数，当然，如果前面已经全量获取过一次数据，可以加上resourceVersion参数，调用方式解决了，剩下的就是如何获取返回，由于是持续获取数据，肯定不能用上面perform()，因此，需要使用CURLOPT_WRITEFUNCTION选项，在lua中则是curl.OPT_WRITEFUNCTION：

``` lua
write_function = function(tab, buffer)
	if type(tab) == "table" then
		table.insert(tab, buffer)
	end
	return #buffer
end

c:setopt(curl.OPT_WRITEFUNCTION, write_function)
```

这里有两个问题需要注意：

* 回调函数的返回值必须是#buffer，当然，如果就是要返回错误并退出则另说，如果返回值的大小跟buffer的长度不等，就会抛出错误CURLE_WRITE_ERROR
* 给定的回调函数的调用时机：当有数据收到时，则立即调用，也就是说，可能数据还没有接收完成，例如，在k8s中调用一次接收到的数据可能不是一个完整的变更，因此，需要自行进行拼接


``` lua
local global_buffer

write_function = function(tab, buffer)
	local ret_val = #buffer

	local obj, info = cjson_safe.decode(buffer)
	if obj == nil then
		global_buffer = global_buffer .. buffer
		obj, info = cjson_safe.decode(global_buffer)
		if obj == nil then
			return ret_val
		else
			buffer = global_buffer
			global_buffer = ""
		end
	end

	-- obj就是一次完整变更的json对象

	return ret_val
end
```

另一个要注意的选项就是CURLOPT_TIMEOUT，它是整个http操作执行的超时时间，大多数场景下，为了防止请求时间过长，例如，网络异常等，因此，都需要设置超时时间，但是，如果对watch操作设置该值，就会导致在watch过程中连接被中断，因此，当进行watch时，该参数需要设置为0，表示不会超时，当然，这也是默认值。

### 2 解析json

lua中使用lua-cjson处理json数据：

``` lua
local cjson = require "cjson"

local arr = {1,2,3}
print(cjson.encode(arr))
```