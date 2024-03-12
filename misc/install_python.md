## python环境搭建(Python 3.11.8)

### 1 安装Python

按照[linux安装python3](https://blog.csdn.net/Yaphets_dan/article/details/129441949)文档安装python3，只是在编译python3时添加一个选项：`--enable-shared`，该选项是为了后续可以将程序打包为二进制，如果不需要该功能，也可以不加。

如果编译python3时加上了`--enable-shared`，需要在安装完成后执行以下命令：

```
echo "/usr/local/python3/lib/" >> /etc/ld.so.conf
ldconfig
```

### 2 使用虚拟环境

使用python进行开发的同学有个体会，如果多人在同一个python环境进行开发，上面有其他同学安装的很多包，我可能用不上，但是上面又没有我需要的安装包，此时，我需要使用`pip install`命令安装我需要的包，但是开发完成后通常不会执行`pip uninstall`进行卸载，可能有其他人用了，或者我自己还要用，所以，会导致机器上安装了大量的包，但是其实在一段时间内用的包可能很少，也没有人敢卸载，而且不同开发者在使用这个环境时也无法生成`requirements.txt`，因为没办法知道具体有哪些包是我用的。

因此，python提供了一种虚拟环境的机制，相当于克隆一个当前的python环境，但是里面没有安装任何包（也可以克隆当前python环境中已经安装的包），在早期版本中是需要额外安装`virtualenv`的包，在比较新的版本中是可以直接使用内部的模块安装的。

`python3 -m venv myenv`：在当前目录下创建一个myenv的目录，里面就是拷贝当前环境python得到的文件。

`. ./myenv/bin/activate`：激活虚拟环境，此时会在bash提示符的前面显示虚拟环境的名称`(myenv)`。

此时，我们就身处于myenv这个虚拟环境中，使用`pip list`会发现只有默认的pip和setuptools，然后我们就可以自由的在里面进行开发了，也可以使用`pip freeze`生成`requirements.txt`。

`deactivate`：该命令可以让我们退出虚拟环境。

### 3 Python程序打包为二进制

当开发完成后，需要将我们的程序分发，一种方式是直接给python脚本，这种方式其他人就可以看到你的逻辑，但是对于不想知道你的逻辑，只是想调用的话，就需要安装python的环境，但是大家知道，python的环境跟python的版本和包的版本相关，有时候还比较麻烦，另一种方式是将程序编译为二进制，其他人不关心你的实现逻辑，只想调用的话，这种方式是比较好的。

`pip install pyinstaller`安装pyinstaller包，然后对我们的主程序进行打包：`pyinstaller --onefile main.py`(此处假设入口脚本是main.py)，然后就会在当前目录下生成`dist`目录，里面就有生成的可以直接调用的main二进制。
