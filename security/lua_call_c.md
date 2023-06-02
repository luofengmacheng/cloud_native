## lua如何调用C/C++

### 1 lua vs C/C++

lua是脚本语言，优点是门槛低，可以热更新，缺点当然就是性能。C/C++是编译型语言，有点是性能高，但是相对的，门槛高，技术不好的人写的代码可能还没有lua的性能高，容易出现core，不能热更新。

不过，lua语言本身就是用C实现的，而且，可以将很多能力封装成lua的接口供lua调用。

### 2 C/C++如何给lua提供接口

查看一个lua模块的源代码会发现，lua模块的实现中既包含lua代码，也包含C代码，其中，C代码的主要逻辑就是获取参数，调用系统调用，返回值，C代码会编译为so供lua调用，而lua代码就是将C代码提供的一些接口进行再封装，以便在lua中更好用，更简单，然后再通过lua代码对外提供接口。因此，这么看起来，通过lua调用C函数，重要的就是在C中如何获取参数以及如何返回值。

下面的说明以[linotify](https://github.com/luofengmacheng/linotify)项目进行说明。

#### 2.1 lua模块的查找

当在lua里面通过require("inotify")时，lua怎么知道去哪里查找inotify模块呢？此时，inotify模块是个lua脚本还是个so呢？

跟其他脚本语言类似，lua中也是通过变量来控制模块的查找的，其中package.path是搜索lua模块的路径，package.cpath是搜索so模块的路径，先查找lua模块，再查找so模块。

通过上面这种方式，在当前目录找到了inotify.so。

#### 2.2 so的入口

要调用inotify.so中的函数，肯定还是要用动态库的函数：dlopen、dlsym。例如，当调用require("inotify")时，如果没有导入inotify.so，则调用dlopen加载inotify.so库，

当在lua中调用`local wd = handle:addwatch('/home/rob/', inotify.IN_CREATE, inotify.IN_MOVE)`时，会调用handle_add_watch()函数。

#### 2.3 参数获取

lua和C之间是通过栈进行交互的，当调用C函数时，C函数的第一个参数是lua_State的指针，可以将它理解为lua的一个状态机。

如果要调用函数，第一步就是参数的获取，lua会将参数放到栈中，因此，inotify.so中的函数可以获取栈中的数据得到参数：

``` C
    fd = get_inotify_handle(L, 1);
    path = luaL_checkstring(L, 2);
    top = lua_gettop(L);
    for (i = 3; i <= top; i++) {
        mask |= (uint32_t)luaL_checkinteger(L, i);
    }
```

get_inotify_handle()获取栈中的第1个参数，luaL_checkstring(L, 2)获取栈中的第2个参数，且第2个参数是个字符串，然后通过lua_gettop(L)获取所有的参数的个数，再用for循环将剩余的参数通过位或放到mask变量。通过这种方式就分别得到了addwatch()的三个参数。

然后再调用inotify的inotify_add_watch()完成实际的逻辑。

#### 2.5 返回值

当具体的业务逻辑完成后，就需要将返回值传给lua，依旧是通过入栈的方式。

在这里，调用完inotify_add_watch()就得到某个监听操作的描述符，也需要将这个描述符返回，如果操作成功，调用`lua_pushinteger(L, wd)`将wd返回，如果操作失败，则返回3个值：

``` C
static int handle_error(lua_State *L)
{
    lua_pushnil(L);
    lua_pushstring(L, strerror(errno));
    lua_pushinteger(L, errno);
    return 3;
}
```

第1个是nil，第2个是错误信息，第3个是错误码。

因此，在lua中可以这样来调用：

``` lua
local wd, err_info, errno = handle:addwatch('/home/rob/', inotify.IN_CREATE, inotify.IN_MOVE)
if wd == nil then
    print("ERROR=", err_info)
end
```

同时还需要注意handle_add_watch()函数的返回值，返回值表明了lua中函数返回值的个数。例如这里，成功时，只返回描述符，因此，函数返回值是1，失败时，多了额外的错误信息，因此，函数返回值是3。

#### 2.6 一个小的demo

有了上面的了解，可以实现我们的一个小小的demo。

假设我们要实现一个加法操作，实际的加法操作在C中完成，然后在lua中调用。

``` C
#include <lua.h>
#include <lauxlib.h>

static int handle_add(lua_State *L) {
	int a, b, c;

	a = luaL_checkinteger(L, 1);
	b = luaL_checkinteger(L, 2);

	c = a + b;
	lua_pushinteger(L, c);

	return 1;
}

static luaL_Reg funcs[] = {
	{"add", handle_add},
	{NULL, NULL}
};

int luaopen_demo(lua_State *L) {
	lua_createtable(L, 0, sizeof(funcs)/sizeof(luaL_Reg) - 1);
	luaL_setfuncs(L, funcs, 0);

	return 1;
}
```

那这里的入口函数就是luaopen_demo()，里面就调用了两个函数，先调用lua_createtable创建

将上面的代码编译为so：

``` shell
gcc demo.c -fPIC -shared -o demo.so
```

lua中调用：

``` lua
local demo = require("demo")

print(demo.add(2, 3))
```

### 3 lua FFI

lua C API实现lua的模块使用的是虚拟栈的方式，实现起来太过麻烦，用户需要使用一种新的接口（C API）和模式（虚拟栈）实现，而使用FFI机制，就可以在lua中直接调用C函数。

#### 3.1 一个小例子

``` lua
local ffi = require("ffi")
ffi.cdef[[
int printf(const char*fmt, ...);
]]

ffi.C.printf("hello %s", "world");
```

首先加载ffi模块，然后使用cdef添加C函数的声明，有点类似于C语言中的头文件，然后就可以调用ffi.C中的printf函数。然后就可以使用luajit编译：`luajit hell.lua`。

#### 3.2 调用so

上面的例子是调用C标准库中的函数，如果需要调用其他的so文件呢？

``` C
// libtest.c
#include <stdio.h>

int show(char *str) {
	int ret = 0;
	if(str == NULL) {
		ret = -1;
	} else {
		printf("input: %s\n", str);
	}
	return ret;
}
```

``` shell
# 将上述代码编译为so
gcc -shared -fPIC libtest.c -o libtest.so
```

然后就可以在lua中调用：

``` lua
local ffi = require("ffi")

-- 加载libtest.so
local myffi = ffi.load("test")

-- 声明函数原型
ffi.cdef[[
int show(char *str);
]]

local str1 = "hello"

-- 将字符串类型转换为char*
local str2 = ffi.cast("char *",str1)

-- 调用libtest.so中的show函数
print(myffi.show(str2))
```

### 4 C API vs FFI

FFI相比C API最大的优势就是比较好理解，性能高，但是使用FFI也存在一些兼容性的问题；而C API由于是官方提供的接口，在稳定性方面还是很好的。

### 5 参考文档

* [FFI Tutorial](https://luajit.org/ext_ffi_tutorial.html)