## C++多线程编程(三): std::thread线程类



### 0 前言



在C++11以前，如果要使用多线程，就需要使用各平台的多线程库，例如，Linux上可以使用pthread，windows上可以使用win32或者MFC提供的多线程API，也就是说，如果要使用多线程，就必须与平台绑定，那么程序就不具备跨平台的能力，为了让程序更好的运行在各平台而不需要用大量的宏来控制编译选项，C++11提供了对多线程的支持，提供了std::thread类。



### 1 std::thread的基本使用



从线程的使用流程上，线程必然存在以下操作：



* 创建线程

* 线程运行

* 等待线程结束



因此，一个简单的使用std::thread进行线程编程的代码如下：



``` C++

#include <iostream>

#include <thread>

#include <atomic>

#include <unistd.h>



std::atomic<bool> exit_flag(false);



void func() {

    while(!exit_flag) {

        std::cout << std::this_thread::get_id() << std::endl;

        sleep(1);

    }

}



int main() {

    std::thread th(func);

    sleep(3);

    exit_flag = true;

    th.join();

}

```



* 创建线程：在主函数中创建一个std::thread对象，并使用func函数作为构造函数的参数，此时相当于创建了一个线程，这个线程已经开始执行了

* 线程运行：func函数作为std::thread对象的构造函数的参数，也就是说创建的线程执行的代码就是func函数，func函数的逻辑就是while循环打印线程ID，打印的间隔是1秒，while循环的退出条件是exit_flag=true

* 等待线程结束：线程作为一个独立运行的单元，在进程结束之前必须要等所有的线程正常退出，不然可能出现资源泄露或者数据不一致等问题，因此，这里先设置线程退出的标志，然后用std::thread的join()方法等待线程退出



这里面的关键点是：



* 线程是一个独立的执行单元，在创建线程时需要指定线程的执行函数

* 进程退出时需要让所有的线程安全退出，需要结合退出标志和join()函数来让线程正常退出，保证资源的正确回收



### 2 std::thread的线程创建和传参



首先还是需要提到线程创建。前面在创建线程时，给的参数是个函数，但是从代码执行角度来说，`任何可以执行`的元素都可以放到这里，例如：



* Lambda表达式

* 函数对象

* 非静态成员函数

* 静态成员函数



不过在实际开发过程中，一般不会使用Lambda表达式、函数对象，使用函数和成员函数居多。



而且，既然是函数，是可以传递参数的，也就是说，在创建线程时，可以给执行的线程的方法传递参数：



``` C++

#include <iostream>

#include <thread>

#include <unistd.h>



void func(int cnt) {

    while(cnt--) {

        std::cout << std::this_thread::get_id() << std::endl;

        sleep(1);

    }

}



int main() {

    std::thread th(func, 3);

    th.join();

}

```



这里去掉了线程退出的标志，而是给func函数传递一个参数，用于控制while循环执行的次数。



### 3 std::thread的其他相关方法



std::thread除了提供创建和等待退出的方法外，还提供了其他功能：



* get_id()：获取线程ID，这个ID用于区分不同的线程

* operator=：线程赋值，不同的线程对象代表不同的线程，因此，线程对象只有移动语义，没有拷贝语义，也就是只能用`std::thread th=std::thread(func)`，而不能用`std::thread th=th2`

* joinable/join：线程是否可以等待退出

* detach：线程分离

* swap/std::swap()：线程交换



这些方案里面比较不好理解的是joinable和detach，而这两个函数刚好是有关联的。



可以将线程理解为前台线程和后台线程，默认创建的线程是前台线程，此时线程对象执行joinable()时返回true，且可以对线程对象执行join()，当线程对象执行detach()后，线程变为后台线程，此时线程对象执行joinable()时返回false，此时不能执行join()，如果对后台线程执行join()会抛出std::system_error异常。



因此，如果不需要等待线程结束或者不关心线程后续执行情况，就可以对线程执行detach()，然后使用变量控制线程的结束；如果需要使用join()等待线程结束，就不能对线程执行detach()，然后结合变量和join()等待线程结束。由于通常需要对线程执行join()，保证在进程退出时线程也正常退出，因此，尽量不对线程对象执行detach()。



### 4 std::this_thread的使用



std::this_thread是包含一些访问当前线程的函数的namespace，包含以下4个函数：



* get_id()：获取线程ID，在其他线程中可以使用std::thread.get_id()获取某个线程的ID，而在当前线程中则可以使用std::this_thread::get_id()获取线程ID

* yield()：当前线程放弃执行，重新进行调度

* sleep_until/sleep_for：当前线程休眠一段时间



### 5 总结



* C++11提供了多线程类std::thread，进行了更高层次的抽象，屏蔽了不同平台的线程实现机制，让我们的程序移植性更高，不需要在代码中用宏进行调用函数的控制

* std::thread多线程类的使用方式与之前用pthread没有很多不同，也提供了创建线程、等待线程退出等机制，构造函数可以用任何可以执行的元素作为参数，也可以给执行元素传递参数

* 线程可以分为前台线程和后台线程，默认创建的是前台线程，如果不关心线程的后续执行情况，可以使用detach()让线程变成后台线程，但是后续就不能调用join()，否则会抛出std::system_error异常

* std::this_thread中提供了操作当前线程的函数，常用的有get_id()和yield()，分别用于获取线程ID和重新调度