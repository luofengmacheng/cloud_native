## stm32之使用串口下载程序

### 1 stm32常用的程序下载方式

将程序下载到stm32芯片通常有以下三种方式：

* 串口转USB：需要用到一个CH340芯片，网上购买stm32基础套件时一般会有使用该芯片连接USB的模块，连接该芯片时需要占用芯片的USTART_TX和USTART_RX，分别对应芯片的PA9和PA10
* SWD方式：例如，st-link v2，需要使用stm32的SWDIO和SWCLK引脚，分别对应芯片的PA13和PA14
* J_LINK方式：需要使用stm32的TCK(时钟)、TMS(模式选择)、TDI(数据输入)、TDO(数据输出)、TRST(复位)引脚

### 2 stm32的启动配置

启动配置就是指定程序开始执行的位置。由于stm32芯片内部有多种存储设备，可以让程序从不同的地方启动，因此，需要对启动的存储设备进行配置。

stm32可以通过配置BOOT0和BOOT1引脚设置启动配置：

* BOOT1=x，BOOT0=0：最常用的模式，执行Flash程序存储器中的程序
* BOOT1=0，BOOT0=1：串口下载模式，执行系统存储器中的程序，该程序是一段BootLoader程序，会将串口发送来的数据写入到Flash程序存储器
* BOOT1=1，BOOT0=1：内置SRAM模式，一般用于调试，较少使用

按照以上说明，如果要使用串口下载程序，就需要先使用串口下载模式将程序下载到Flash程序存储器，然后使用常规模式执行Flash程序存储器中的程序。

启动配置只在上电复位时有效，也就是说，只有当通电和复位时会根据BOOT0和BOOT1确定启动配置。

### 3 使用串口下载程序

使用串口下载程序之前需要将CH340的引脚接好，具体连线为：

* A9 -> RXD
* A10 -> TXD
* GND -> GND
* VCC -> 3.3V

然后将USB TO TTL插入电脑的USB接口，并将电源线接好。

将程序编译为HEX文件后，就可以将程序下载到stm32中。

这里使用[FlyMcu](http://www.mcuisp.com/chinese%20mcuisp%20web/ruanjianxiazai-chinese.htm)软件将HEX文件写入到stm32中，从上述官网下载程序包，直接解压就可以运行：

![FlyMcu](https://github.com/luofengmacheng/cloud_native/blob/master/stm32/pics/flymcu.jpg)

先设置为串口下载模式(BOOT1=0，BOOT0=1)，然后按下复位(如果是刚上电，可以不用按复位)，再点击“开始编程”，程序就会下载到stm32中开始执行。

* 当BOOT1=0、BOOT0=1时，如果按复位，此时就会执行BootLoader程序等待串口数据写入，当数据写入Flash程序存储器后，还是会执行Flash程序存储器中的程序
* 当BOOT1=x、BOOT0=0时，如果按复位，此时还是会执行Flash程序存储器中的程序

因此，如果是开发，可以直接使用串口下载模式，每次编译好HEX文件后，点击复位，然后下载程序，该过程就跟C51单片机下载过程类似了。

### 4 总结

当从网上购买stm32开发板基础套件时，可能只有USB TO TTL，因此只能使用串口下载方式。使用串口下载方式需要了解stm32的启动配置，也就是BOOT0和BOOT1对应的含义。

下载程序之前需要先将CH340的引脚接好，然后在点击复位时，将程序下载到stm32中。
