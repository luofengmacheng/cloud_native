## VsCode调试C++

### 1 配置编译任务

使用VsCode进行C++开发时，除了在机器上安装必要的编译工具（例如，gcc、g++、cmake等）之外，还需要在VsCode配置编译任务，从而可以通过点击或者快捷键的方式进行快速编译。

配置编译任务需要配置两个文件：

* c_cpp_properties.json：环境管理
* tasks.json：编译任务

配置c_cpp_properties.json：

ctrl + p，输入> C/C++，在其中找到Edit Configuration：

![Edit Configuration](https://github.com/luofengmacheng/cloud_native/blob/master/security/pics/debug_edit_configurations.jpg)

需要配置的项主要有：

* Compiler path：编译器路径
* C standard：C语言版本
* C++ standard：C++版本

配置tasks.json：

ctrl + p，输入> tasks：

![Configure Default Build Task](https://github.com/luofengmacheng/cloud_native/blob/master/security/pics/debug_configure_default_build_task.jpg)

然后将json修改为：

``` json
{
	"version": "2.0.0",
	"tasks": [
		{
			"type": "shell",
			"label": "cmake",
			"command": "cmake",
			"args": [
				"-G",
				"Unix Makefiles",
				"-DCMAKE_BUILD_TYPE=Debug"
			],
			"group": "build",
			"problemMatcher": [],
			"detail": "CMake template build task"
		},
		{
			"label": "make",
			"command": "make",
			"problemMatcher": []
			
		},
		{
			"label": "Build",
			"dependsOrder": "sequence",
			"dependsOn": [
				"cmake",
				"make"
			],
			"group": {
				"kind": "build",
				"isDefault": true
			}
		}
	]
}
```

tasks中有三个对象，第一个是执行cmake命令，第二个是make命令，第三个是将第一个和第二个整合起来，顺序调用cmake和make，并且设置为默认操作，因此，当选择`Terminial->Run Build Task`时，就会依次执行cmake和make进行编译。

### 2 配置调试任务

ctrl + p，输入> Debug：

![Debug: Add Configuration](https://github.com/luofengmacheng/cloud_native/blob/master/security/pics/debug_add_configuration.jpg)

随便选择一个应用类型，然后将launch.json修改为：

``` json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "type": "cppdbg",
            "request": "launch",
            "name": "gdb",
            "program": "${workspaceFolder}/test",
            "MIMode": "gdb",
            "cwd": "${workspaceFolder}"
        }
    ]
}
```

* type：类型
* name：名称，用于区分多个配置
* program：调试的程序
* MIMode：指定调试器，这里用gdb
* cwd：当前目录，可以直接使用变量

然后就可以打开调试窗口启动调试：

![Run and Debug](https://github.com/luofengmacheng/cloud_native/blob/master/security/pics/debug_windows.jpg)

### 3 进行调试

在调试窗口，可以设置断点，然后进行调试，调试过程中可能会使用到4个快捷键：

* F10（Step Over）：一步一步执行
* F11（Step Into）：进入函数执行
* F5（Continue）：继续执行，在下一个断点处停住
* Ctrl + Shift + F5（Stop）：停止调试

![Debug](https://github.com/luofengmacheng/cloud_native/blob/master/security/pics/debug_work_section.jpg)