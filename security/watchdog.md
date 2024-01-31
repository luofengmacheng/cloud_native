## Linux中的watchdog机制

### 1 为什么需要watchdog

Linux内核代码庞大且复杂，有时候

### 2 如何查看对应的日志

### 3 代码分析

watchdog的初始化函数为lockup_detector_init，它主要做两件事：

* 设置检测阈值
* 如果启用watchdog，则调用watchdog_enable_all_cpus启动所有cpu的watchdog线程

* watchdog_enable_all_cpus
    * smpboot_register_percpu_thread 对于每个CPU，调用__smpboot_create_thread创建线程
    * smpboot_update_cpumask_percpu_thread

``` C
static struct smp_hotplug_thread watchdog_threads = {
	.store			= &softlockup_watchdog,
	.thread_should_run	= watchdog_should_run,
	.thread_fn		= watchdog, // 线程执行的函数
	.thread_comm		= "watchdog/%u", // 线程名
	.setup			= watchdog_enable, // 设置watchdog的定时函数
	.cleanup		= watchdog_cleanup,
	.park			= watchdog_disable,
	.unpark			= watchdog_enable,
};
```

每个CPU都会根据watchdog_threads对象创建一个线程，watchdog_threads对象需要关注两个成员：

* thread_fn=watchdog：线程执行的函数
* setup=watchdog_enable：watchdog的定时函数

在watchdog_enable中会创建一个高精度的定时器去执行watchdog_timer_fn，然后将watchdog_touch_ts设置为当前时间戳，设置当前的优先级为最高。而在watchdog_timer_fn中，

