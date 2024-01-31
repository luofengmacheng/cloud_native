## C++多线程编程(二)：条件变量

### 0 前言

互斥锁是为了保证多个线程在访问共享资源时不会出现不可预期的结果，能够让多个线程不会同时执行lock和unlock之间的代码，也就是说，互斥锁只是保证在访问共享资源时不会出现问题，但是，有一种场景是需要线程之间进行协作，典型的是生产者-消费者模型：生成者生成数据，放到队列后，通知消费者，消费者接收到信号后，从队列中取出数据进行处理。

### 1 生产者-消费者模型

一句话描述生产者和消费者的功能：

* 生产者：生成数据，放到队列
* 消费者：从队列取出数据，对数据进行处理

``` C++
#include <iostream>
#include <thread>
#include <mutex>
#include <queue>
#include <unistd.h>

std::mutex mtx;
std::queue<int> que;
bool comsumer_exit = false;

void Producer() {
    int val = 0;
    int cnt = 10;
    while(cnt--) {
        val = time(nullptr);
        {
            std::unique_lock<std::mutex> lk(mtx);
            que.push(val);
        }
        std::cout << "Producer: push " << val << std::endl;
        sleep(1);
    }
}

void Comsumer() {
    int val = 0;
    while(!comsumer_exit) {
        if(!que.empty()) {
            {
                std::unique_lock<std::mutex> lk(mtx);
                val = que.front();
                que.pop();
            }
            std::cout << "Comsumer: pop " << val << std::endl;
        }
    }
}

int main ()
{
    std::thread pro(Producer);
    std::thread com(Comsumer);
    pro.join();
    std::cout << "Producer exit" << std::endl;
    comsumer_exit = true;
    com.join();
    std::cout << "Comsumer exit" << std::endl;

    return 0;
}
```

上面用了两个线程，一个是生产者线程，生产者使用time()函数获取当前时间戳，然后将时间戳放到队列中，并打印放入队列的数据；另一个是消费者线程，消费者会先判断队列是否为空，如果不为空，则获取对头元素，然后将对头移除，并打印获取到的数据。

以上代码有四个问题：

* 为了让生产者和消费者线程都能够在执行一段时间后退出，对于生产者，只生产10个元素，并且为了降低生产数据的速率，在生产一个元素后休眠1秒，对于消费者，则通过退出标志决定是否退出数据处理循环，此处退出循环后不应该立即退出线程，应该将队列中的数据清理掉，有可能队列中的数据是需要额外进行释放的，这里是基本数据类型，所以没啥问题，如果是复杂数据类型，且是new出来的，就需要进行释放了。
* 为了对队列的互斥访问，在队列的插入和读取过程加上了unique_lock，将数据的打印放在了unique_lock范围之外，这会导致在打印数据时可能出现乱序，因此，这里需要将std::cout语句也放到unique_lock范围内。
* 消费者线程使用退出标志决定是否退出循环，该退出标志也是由两个线程使用的，因此，同样需要进行加锁控制，但是，如果用mutex就会感觉比较别扭，这种变量就很适合使用atomic。
* 这样一个简单的程序，虽然运行起来没有问题，但是会发现运行期间，进程的CPU占用居然达到了100%，这是由于消费者线程在执行过程中会持续调用queue.empty()查看队列是否有元素，只有队列不空时才会从队列中取出元素，但是队列大部分时间都是空的，因此该线程大部分时间都在执行queue.empty()，虽然该逻辑很简单，就是获取底层对象保存的数据量，但是如果一直在执行，从而导致`用户态CPU`占比持续100%

![High User CPU](https://github.com/luofengmacheng/cloud_native/blob/master/security/pics/condition_variable.jpg)

前三个问题比较好解决，这里重点看下第四个问题，问题的主要原因在于，线程不知道队列是否有数据，只能通过不断轮询的方式判断队列是否有数据。

### 2 通过sleep降低用户态CPU

既然问题的原因是轮询的方式导致用户态CPU占比高，那最简单的版本就是降低轮询的频率。

在消费者线程的que.empty()的if语句处添加else，让线程睡眠1秒，会发现用户态CPU降为0%。这种方式是让生产者线程和消费者线程的频率一致，那么，在消费者每次执行que.empty()时都为false，也就是没有空轮询的情况。而且，也能发现，如果将1秒改成0.5秒，用户态CPU还是100%，而将1秒改成2秒时，用户态CPU就会降为0，但是消费效率降低了。

这种方式的问题在于，实际开发过程中，消费者线程不可能预知生产者线程的生产频率，而且，生产者的生产频率也不一定是固定的，可能有时候快，有时候慢，或者有一段时间就是没有生产数据，因此，这种通过sleep进行休眠的方式在实际中是不可行的，要么没有作用，要么会拉低性能。

我们需要的是一种通知机制，当生产者将数据放到队列时`通知`消费者：嘿，队列中有数据了。消费者`收到`消息，就可以从队列中取出数据，当生产者没有生产数据时，消费者线程就休眠，不会消耗用户态CPU，条件变量就是这样一种机制。

### 3 条件变量：std::condition_variable

条件变量有两类方法：

* wait、wait_for、wait_until：等待满足条件，如果不满足条件则进入休眠状态，wait_for和wait_until与互斥量中的timed_mutex的lock_for和lock_until类似，只会等待一段时间
* notify_one、notify_all：通知满足条件，notify_one只会唤醒一个线程，而notify_all会唤醒等待的所有线程

wait和notify实现的伪代码如下：

``` C++
std::queue<std::thread> waitingThreads; // 等待队列
boolean conditionMet; // 条件是否满足的标志

void wait(std::mutex mtx) {
    // 将当前线程添加到等待队列
    waitingThreads.push_back(current_thread());

    mtx.unlock();

    if(!conditionMet) {
        try {
            wait();
        } catch (InterruptedException e) {
            // 处理中断异常
        }  
    }

    mtx.lock();
}

void notify() {
    // 更新条件标志
    conditionMet = true;

    // 唤醒等待队列中的第一个线程
    if (!waitingThreads.empty()) {
        std::thread th = waitingThreads.front();
        waitingThreads.pop();
        th.interrupt(); // 发送中断信号
    }
}
```

wait：先将线程加入等待队列，在检查条件之前释放互斥锁，然后检查条件是否满足条件，如果不满足，当前线程进入休眠状态，当线程被唤醒时，再次进行加锁

notify：先更新条件标志，然后检查等待队列中是否有线程，如果有，则唤醒队列中的第一个线程

因此，从程序执行的角度看，一个线程执行wait时会进入休眠状态，该线程会被放到条件变量的等待队列中，另一个线程在执行一些操作并执行notify_one后，会唤醒条件变量的等待队列中的一个线程，之前的线程被唤醒后继续执行，但是这里存在两个问题：

* 虚假唤醒：线程被唤醒和条件检查操作不是原子操作，存在线程被唤醒，但是条件并不满足的情况，因此，当线程被唤醒后，也就是wait之后应该再次检查条件是否满足
* 信号丢失：如果生产者线程在消费者线程还没有执行wait时就发送信号，则信号可能丢失，实际开发过程中，由于线程的执行顺序是无法预知的，不太可能让wait操作先执行，但是可以在wait之前先检查条件是否满足

``` C++
#include <iostream>
#include <thread>
#include <mutex>
#include <queue>
#include <condition_variable>
#include <unistd.h>

std::mutex mtx;
std::condition_variable cv;
std::queue<int> que;
bool comsumer_exit = false;
std::unique_lock<std::mutex> lk(mtx);

void Producer() {
    int val = 0;
    int cnt = 10;
    while(cnt--) {
        val = time(nullptr);
        {
            std::unique_lock<std::mutex> lk(mtx);
            que.push(val);
            std::cout << "Producer[" << std::this_thread::get_id() << "]: push " << val << std::endl;
            cv.notify_one();
        }
        sleep(1);
    }
}

void Comsumer() {
    int val = 0;
    while(!comsumer_exit) {
        std::unique_lock<std::mutex> lk(mtx);
        // 如果被唤醒，则检查是否满足条件
        // 同时，在wait之前先判断条件是否满足
        while(que.empty() && !comsumer_exit) {
            cv.wait(lk);
        }
        if(!que.empty()) {
            val = que.front();
            que.pop();
            std::cout << "Comsumer["<< std::this_thread::get_id() << "] : pop " << val << std::endl;
        }
        else {
            std::cout << "queue is empty" << std::endl;
        }
    }
}

int main ()
{
    std::thread pro(Producer);
    std::thread com(Comsumer);
    pro.join();
    std::cout << "Producer exit" << std::endl;
    comsumer_exit = true;
    cv.notify_one();
    com.join();
    std::cout << "Comsumer exit" << std::endl;

    return 0;
}
```

上述代码将条件判断放在wait之前，其实wait操作还提供第二个参数，将条件判断交给wait来做，在唤醒之后判断条件是否满足：

``` C++
#include <iostream>
#include <thread>
#include <mutex>
#include <queue>
#include <condition_variable>
#include <unistd.h>

std::mutex mtx;
std::condition_variable cv;
std::queue<int> que;
bool comsumer_exit = false;

void Producer() {
    int val = 0;
    int cnt = 10;
    while(cnt--) {
        val = time(nullptr);
        {
            std::unique_lock<std::mutex> lk(mtx);
            que.push(val);
            std::cout << "Producer[" << std::this_thread::get_id() << "]: push " << val << std::endl;
            cv.notify_one();
        }
        sleep(1);
    }
}

void Comsumer() {
    sleep(1);
    int val = 0;
    while(!comsumer_exit) {
        std::unique_lock<std::mutex> lk(mtx);
        cv.wait(lk, []{ return !que.empty() || comsumer_exit; });
        if(!que.empty()) {
            val = que.front();
            que.pop();
            std::cout << "Comsumer["<< std::this_thread::get_id() << "] : pop " << val << std::endl;
        }
        else {
            std::cout << "queue is empty" << std::endl;
        }
    }
}

int main ()
{
    std::thread pro(Producer);
    std::thread com(Comsumer);
    pro.join();
    std::cout << "Producer exit" << std::endl;
    comsumer_exit = true;
    cv.notify_one();
    com.join();
    std::cout << "Comsumer exit" << std::endl;

    return 0;
}
```

最后，需要解释下条件变量使用过程中对互斥量的使用。从代码理解的角度上看，互斥量可以保护条件变量的并发使用，但是使用互斥量的主要原因是保护队列的并发访问以及条件的判断，如果不加锁，消费者线程被唤醒，以及检查条件变量不是原子操作，有可能被唤醒后被调度到其他线程，然后再回来执行时条件又不满足了，造成虚假唤醒的情况。

如果将生产者的notify操作放到互斥量外面，可能导致其他一些问题，例如，当调用notify时，另一个线程可能正在检查条件变量，于是出现在未通知的情况下线程被唤醒。

### 4 总结

* 互斥量是用于解决多个线程同时访问某个资源的并发问题，条件变量是解决多个线程之间进行同步的问题
* 典型的生产者-消费者模型中，在消费者线程中使用轮询方式查看队列是否有元素的方式，会导致线程的用户CPU占比高，通过sleep的方式只能在某些场景下有效
* 条件变量提供了wait和notify方法用于两个线程之间进行同步，使用过程中需要注意虚假唤醒和信号丢失的问题，同时要注意条件变量需要与互斥量一起使用，并且wait和notify都需要放到互斥量的临界区中
