## C++多线程编程(一)：互斥锁

### 0 前言

在现代程序开发中，会大量使用多线程机制，很多语言都内置了对多线程的支持，而C++直到C++11才提供了对多线程的支持，既然支持多线程，那么一定也提供了锁的支持。

为什么多线程就一定用锁呢？因为当程序以多线程运行时，如果有对共享资源的使用，例如，两个线程同时对共享变量进行修改，由于这些操作不是原子操作，就会导致出现异常情况，修改的两个线程都认为操作成功了，但是实际上只有一个成功了。这时就需要锁去保证两个操作都分别执行完成。

因此，使用锁就需要搞清楚要保护的共享资源。

### 1 互斥锁：std::mutex

互斥锁是最常见的锁，通常提供两个基本操作：

* 申请锁：Lock，如果锁当前未被占用，则占用锁，然后执行其他代码
* 释放锁：Unlock

``` C++
int shared_val = 1;
std::mutex mtx;

// thread1
mtx.lock();
shared_val += 1;
mtx.unlock();

// thread2
mtx.lock();
shared_val -= 1;
mtx.unlock();
```

两个线程都对共享变量shared_val进行操作，第一个线程对变量加1，第二个线程对变量减1，如果不加锁，最后的结果可能是0，也可能是-1，也可能是1，加锁后才能保证最终的shared_val为0。

执行lock()操作时，如果锁已经被其他线程占用，则当前线程会进入休眠状态，并加入到锁的等待队列中，当锁被释放时，则被唤醒。如果不希望线程进入休眠状态，可以使用try_lock()，相当于是lock()的非阻塞版本：

``` C++
// thread2
if(mtx.try_lock()) {
    shared_val -= 1;
    mtx.unlock();
}
```

执行try_lock()时，会尝试占用锁，如果占用锁成功，则执行操作，然后释放锁。

### 2 递归锁：std::recursive_mutex

递归锁是为了处理一种特殊的场景：线程在占用锁的情况下再次申请锁。

如果使用mutex的话，就会造成死锁，因为在mutex看来，锁已经被占用了，于是会进入休眠状态，但是锁永远不可能被释放，因为是自己占用的。

``` C++
int shared_val = 1;
std::mutex mtx;

void func() {
    mtx.lock();
    mtx.unlock();
}

// main
mtx.lock();
func();
mtx.unlock();
```

在主函数中，先加锁，而在加锁的逻辑中调用func()函数，func()函数则再次进行加锁，于是就会造成死锁。注意：如果直接执行上述代码，如果将下面的加锁代码放到std::thread中就会出现死锁。

为了解决递归加锁的问题，可以将上面的std::mutex改成std::recursive_mutex即可，两种类型的锁的用法一样，只是递归锁可以重复加锁而不会造成死锁。

### 3 时间锁：std::timed_mutex

std::timed_mutex是带有时间的mutex，也就是说在申请锁时，如果锁已经被占用，可以等待一段时间，如果还是被占用，则不继续等待，因此，与std::mutex不同的是，std::timed_mutex有两个与时间相关的方法：

* try_lock_for：尝试申请锁，参数为最多等待的时间段
* try_lock_until：尝试申请锁，参数为等待结束的时间点

``` C++
#include <iostream>       // std::cout
#include <chrono>         // std::chrono::milliseconds
#include <thread>         // std::thread
#include <mutex>          // std::timed_mutex

std::timed_mutex mtx;

void fireworks () {
  // 申请锁，最多等待200ms
  while (!mtx.try_lock_for(std::chrono::milliseconds(200))) {
    std::cout << "-";
  }
  // 获取锁之后，休眠1s
  std::this_thread::sleep_for(std::chrono::milliseconds(1000));
  std::cout << "*\n";
  mtx.unlock();
}

int main ()
{
  std::thread threads[10];
  // 创建10个线程
  for (int i=0; i<10; ++i)
    threads[i] = std::thread(fireworks);

  for (auto& th : threads) th.join();

  return 0;
}
```

与recursive_mutex类似，timed_mutex也有对应的recursive_timed_mutex。

关于递归锁的使用：建议不要使用递归锁，递归锁可能会掩盖一些问题。

### 4 RAII(lock_guard & unique_lock)

RAII是资源申请即初始化，是一种编程技术，在构造函数中获取资源，在析构函数中释放资源，通过这种方式保证资源正确的被管理。

以文件操作为例：

``` C++
#include <iostream>
#include <fstream>

int main() {
    std::ofstream file("example.txt");
    if (!file.is_open()) {
        std::cerr << "Failed to open file" << std::endl;
        return 1;
    }

    file << "Hello, world!";
    file.close();

    return 0;
}
```

上述代码就是未采用RAII的方式，直接用流的方式打开文件，然后向文件中写入数据，最后关闭文件，如果写入数据中间发生错误，或者中间其他逻辑出现异常，就可能导致未成功关闭文件。而使用RAII时，就可以将文件打开的操作放到类的构造函数，将文件关闭的操作放到析构函数，那么只要退出当前作用域就会调用析构函数，即关闭文件。

因此，使用RAII通常就涉及到一种资源，典型的应用场景有：

* 动态内存分配：当使用new分配内存时，在析构函数中释放内存
* 文件管理：在构造函数中打开文件，在析构函数中关闭文件
* 线程管理：在析构函数中释放线程资源
* 锁的管理：在析构函数中释放锁
* 数据库连接：在析构函数中释放连接

可以看出，这些场景都是为了保证在退出作用域时资源被正确的释放。很显然，锁也是个很常见的场景，C++也提供了对锁的支持。

C++提供了两个RAII风格的包装类：lock_guard和unique_lock，可以将它们的实现理解为，在创建对象时执行lock，在析构对象时执行unlock。那为什么还需要提供两个类呢？它们的区别是什么？

lock_guard用于简单的加锁和释放锁的操作，并且模板类只有构造函数和析构函数。

``` C++
#include <iostream>
#include <thread>
#include <mutex>

std::mutex mtx;
int shared_val = 0;

void fireworks () {
  std::lock_guard<std::mutex> lk(mtx);
  shared_val += 1;
}

int main ()
{
  std::thread threads[100];
  for (int i=0; i<100; ++i)
    threads[i] = std::thread(fireworks);

  for (auto& th : threads) th.join();
  std::cout << shared_val << std::endl;

  return 0;
}
```

上述代码创建100个线程，每个线程中都对共享变量shared_val加1，如果不加锁，最终的结果可能不是100，这里使用的就是lock_guard，而且是在fireworks函数的开始处，也就是说，在创建对象lk时执行mtx.lock()，在退出函数时执行mtx.unlock()，整个函数都处于加锁的范围中，如果要缩小锁的范围，可以使用`{}`缩小变量的作用域。

``` C++
#include <iostream>
#include <thread>
#include <mutex>

std::mutex mtx;
int shared_val = 0;

void fireworks () {
  {
    std::lock_guard<std::mutex> lk(mtx);
    shared_val += 1;
  }
}

int main ()
{
  std::thread threads[100];
  for (int i=0; i<100; ++i)
    threads[i] = std::thread(fireworks);

  for (auto& th : threads) th.join();
  std::cout << shared_val << std::endl;

  return 0;
}
```

通过这种方式就可以将加锁的区域限定在一个小的范围，这基本就是lock_guard的所有用法了。

unique_lock则提供了很多其他功能：

* 加解锁的方法与timed_mutex一致，也就是说unique_lock也可以执行lock进行加锁，执行unlock进行解锁
* 修改unique_lock的方法，operator=(替换对应的mutex)、swap(交换对应的mutex以及状态)、release(断开与mutex的关联关系，也就是不再管理任何mutex，在断开关联关系之前返回mutex的指针)
* 查看占用锁的情况：owns_lock(检查当前锁是否已经执行lock)、operator bool(与owns_lock相同)、mutex(返回对应的mutex的指针)

除了上述方法的区别，另一个区别是构造函数的区别，unique_lock提供了很多构造函数，因此，在创建unique_lock对象时可以指定一些参数。

``` C++
unique_lock() noexcept; // 默认构造函数
explicit unique_lock (mutex_type& m); // 最常使用的构造函数，参数就是一个mutex
unique_lock (mutex_type& m, try_to_lock_t tag); // 第二个参数是个try_to_lock_t类型的tag
unique_lock (mutex_type& m, defer_lock_t tag) noexcept; // 第二个参数是个defer_lock_t类型的tag
unique_lock (mutex_type& m, adopt_lock_t tag); // 第二个参数是个adopt_lock_t类型的tag
template <class Rep, class Period>unique_lock (mutex_type& m, const chrono::duration<Rep,Period>& rel_time); // 第二个参数是个时间段，也就是说在执行lock时只会等待一段时间，类似timed_mutex
template <class Clock, class Duration>unique_lock (mutex_type& m, const chrono::time_point<Clock,Duration>& abs_time); // 第二个参数是个时间点，也就是说在执行lock时只会等到某个时间点，类似timed_mutex
```

其中比较特殊的就是中间的三个构造函数，它们通过提供额外的参数来控制创建的unique_lock的行为：

* try_to_lock_t：可以设置为std::try_to_lock，创建unique_lock对象时不执行mtx.lock()，而是执行mtx.try_lock()，因此，有可能没有加锁成功
* defer_lock_t：可以设置为std::defer_lock，延迟加锁，也就是在创建unique_lock对象时不立即执行mtx.lock()，后续可以再执行lock操作
* adopt_lock_t：可以设置为std::adopt_lock，当前线程已经占用锁，也就是说在创建unique_lock对象之前就已经执行过mtx.lock()

这些标志使得unique_lock可以完成一些更加细粒度的控制操作，可以应用于一些特殊场景。

``` C++
#include <iostream>
#include <thread>
#include <mutex>

std::mutex mtx;
int shared_val = 0;

void fireworks () {
  std::unique_lock<std::mutex> lk(mtx, std::defer_lock);
  lk.lock();
  shared_val += 1;
  lk.unlock();
}

int main ()
{
  std::thread threads[100];
  for (int i=0; i<100; ++i)
    threads[i] = std::thread(fireworks);

  for (auto& th : threads) th.join();
  std::cout << shared_val << std::endl;

  return 0;
}
```

上述代码在创建unique_lock时加上了defer_lock，说明lk对象在创建后没有自动执行mtx.lock()，就需要在后面手动执行lock操作了。

### 5 总结

* 锁是多线程中保证业务逻辑正确运行的一种机制，最常使用的就是互斥锁mutex，它能保护共享资源在并发操作时不会产生未知的结果。
* 为了让锁可以被重复加锁而提供了recursive_mutex。
* 为了让加锁时可以等待一段时间，提供了timed_mutex。
* 锁的操作存在加锁和释放锁的过程，如果锁没有被正确释放，会造成资源泄露以及死锁的可能，为了将锁的使用和对象的声明周期关联，使用RAII机制实现了lock_guard和unique_lock，大部分场景下可以使用lock_guard，如果需要对加解锁过程进行更多控制可以使用unique_lock
