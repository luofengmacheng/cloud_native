## GMP调度模型

### 1 概念

goroutine作为golang中的协程，对golang的`高性能`起着重要作用。协程被称为`用户态的线程`，既然是线程，就需要调度，让该线程有机会执行，并且在适当的时候让出执行资源。

golang中的调度器有2代：

#### 1.1 第1代调度器

go1.1以前的调度器使用的是GM，G代表goroutine，M代表Machine，表示系统线程，当用go语句创建goroutine后，将goroutine放到全局队列，M从全局队列中获取goroutine并执行，如果goroutine由于系统调用或者阻塞就会让出执行资源，M会将goroutine放回全局队列，由于全局队列是所有M共享，因此就需要对全局队列加互斥锁，这就会导致以下问题：

* 每个M在获得G时都需要得到锁，有锁竞争的开销
* 局部性差

#### 1.2 第2代调度器

第2代调度器引入了：

* P，Processor，包含运行goroutine的资源，如果线程想运行goroutine，必须先获取P，P中还包含可运行的G队列
* work stealing，当M绑定的P没有可运行的G时，可以从其他运行的M那里偷取G

### 2 G、M、P

G：Goroutine，运行的协程本身，其中包含要执行的指令、参数和上下文，当在代码中调用go时，就会创建一个G对象，对应操作系统的PCB(Process Control Block，进程控制块)

M：系统线程或者Machine，当创建一个M时，都会创建一个底层线程

P：Processor，当一个P有任务需要处理，需要创建或者唤醒一个系统线程去处理它队列中的任务，P决定同时执行的任务的数量，对应系统的物理CPU，可以通过GOMAXPROCS环境变量或者`runtime.GOMAXPROCS()`修改

调度器的执行过程：

* 程序使用go指令创建一个goroutine，就会创建一个G对象，G保存到P的本地队列(哪个P的本地队列)，如果本地队列满，则保存到全局队列
* P查看本地队列或者全局队列中是否有G对象，如果有，就会唤醒M
* M得到一个G对象，执行G对象，寻找下一个G

P的队列分为本地队列和全局队列，其中，本地队列是无锁的，没有数据竞争问题，处理速度比较高，全局队列是用来平衡不同的P的任务数量，所有的M共享P的全局队列。

### 3 Goroutine的生命周期

* runtime创建M0、G0，然后绑定M0和G0
* 创建GOMAXPROCS的P列表
* 执行main()函数创建主goroutine(G1)，然后放到P的本地队列
* 启动M0，M0绑定P
* 根据goroutine的栈和调度信息，M0设置运行环境
* 在M中运行G，执行结束
* runtime.main()调用defer，panic，调用runtime.exit()

### 4 抢占式调度

Go1.12实现了请求式抢占调度，调度器在启动时会启动一个单独的线程sysmon，它负责所有的监控工作，当sysmon发现某个goroutine满足以下条件中的一个时就会发出抢占请求，至于goroutine何时停下就不知道了：

* goroutine执行系统调用超过20us
* goroutine运行超过10ms