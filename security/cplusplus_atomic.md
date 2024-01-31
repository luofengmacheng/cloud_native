## C++多线程编程(四): atomic原子操作



### 0 前言



原子操作的意思是该操作执行过程中不能被中断，该操作要么不执行，要么全部执行，不存在执行一部分的情况。在编程语言中，有些操作虽然看起来只有一行，但是变成机器语言后就是多个操作步骤，其中的每个操作步骤都是一个原子操作，但是这些操作合起来却不是原子操作，这样的代码在并发执行时可能会调度到其他线程，从而出现中断的情况，造成数据不一致。



### 1 非原子操作存在的问题



``` C++

#include <iostream>

#include <thread>



int cnt = 0;

 

void func() {

    int c = 10000;

    while(c--) {

        ++cnt;

    }

}

 

int main(){

    std::thread th1(func);

    std::thread th2(func);



    th1.join();

    th2.join();

    std::cout << cnt << std::endl;



    return 0;

}

```



两个线程同时对共享变量cnt执行1万次自增操作，然后在主函数中打印共享变量cnt的值，如果多次执行该代码，会发现最终的结果可能不是2万。虽然`++cnt`看起来只有一行代码，但是在翻译为机器代码时会分成多个步骤：先将cnt放到eax寄存器，然后对eax寄存器加1，最后再将eax寄存器结果放回到cnt。如果多个线程同时执行，这三个操作可能会被打断造成最终的结果错误。



### 2 原子操作的基本使用



为了解决并发执行的非原子操作造成的数据不一致问题，一种方式是可以加锁，例如用std::mutex对上面的++cnt加锁。另一种方式是使用C++11引入的atomic原子类型，原子类型提供的各种操作都是原子的。



``` C++

#include <iostream>

#include <thread>

#include <atomic>



std::atomic<int> cnt(0);

 

void func() {

    int c = 10000;

    while(c--) {

        ++cnt;

    }

}

 

int main(){

    std::thread th1(func);

    std::thread th2(func);



    th1.join();

    th2.join();

    std::cout << cnt << std::endl;



    return 0;

}

```



上述代码与之前的代码只有两个不同点：



* 引入atomic头文件

* 将基本的int类型换成std::atomic<int>类型，并使用0进行初始化



多次运行会发现输出的结果一定是2万。



std::atomic本身是个模板类型，内部保存了基本数据类型的数据，原子对象保证所有的方法在操作数据时不被中断。



### 3 原子操作的方法



std::atomic提供了两大类操作，一类是对所有原子类型的通用方法，另一类是只对特定的原子类型适用的方法。



通用方法：



* store：更新原子对象的数据

* load：获取原子对象的数据

* is_lock_free：是否支持原子操作

* exchange：将原子对象的数据与另一个数据进行交换，返回值是交换前原子对象的数据

* compare_exchange_weak：传入期望值和设定值，将原子对象的数据与期望值进行比较，如果相等，则将原子对象的数据赋值为设定值，如果不相等，则将原子对象的数据更新到期望值

* compare_exchange_strong：与compare_exchange_weak类似，并且可以防止compare_exchange_weak在意外情况下返回false的情况



特定方法：



* fetch_add/fetch_sub：对原子对象的数据进行加和减(整数)

* fetch_and/fetch_or/fetch_xor：位运算(整数)

* operator++/operator--：自增和自减(整数)

* 复合赋值运算：+=、-=(整数和指针)；&=、|=、^=(整数)



总的来说就是：所有的原子对象都支持对内部保存的数据的读取和更新；对于整数的原子对象，支持与常规整数相同的复合赋值运算以及自增和自减。



上述操作里面大部分都是数据读写和对整数的常规操作，比较特别的只有is_lock_free和compare_exchange_weak/compare_exchange_strong。



is_lock_free用于判断当前平台是否可以使用原子操作，因为std::atomic的实现是基于底层硬件的，只有当底层硬件实现了操作的原子性，才能使用std::atomic。但是当前大部分环境都支持原子操作，因此，通常不需要执行该函数。



compare_exchange_weak/compare_exchange_strong则用于实现CAS：CAS(Compare And Swap)是一种更新变量的机制，常用于实现无锁数据结构和算法。具体使用场景为：首先获取当前字段值，然后进行操作，在需要更新时调用compare_exchange，compare_exchange会检查当前值与之前得到的值是否相同，如果相同，说明这段时间(获取当前字段值和更新字段值这段时间)内没有其他线程修改，可以安全的进行更新，否则，就只能重新获取当前的值再次更新。kubernetes里面有类似的机制：每个资源对象的metadata都有一个resourceVersion属性，每次资源更新时，该属性就会自增，当客户端需要更新资源对象，在提交yaml到服务端时，服务端会检查客户端提交的资源更新的resourceVersion，如果跟服务端保存的相同，说明资源没有被更新过，本次可以安全的更新，如果跟服务端保存的不同，说明从上次获取到本地变更之间资源已经被修改了，不能被更新。通常情况下建议使用compare_exchange_strong，虽然它的性能比compare_exchange_weak稍低，但是它提供了更强的数据一致性保证。



### 4 atomic_flag



除了std::atomic，atomic中还提供了另一种简单的原子布尔类型：atomic_flag，而且它只提供两个方法：test_and_set、clear。



atomic_flag是原子布尔类型，那么可以理解为，atomic_flag只有两个状态：true和false。



* test_and_set：检查标志位，如果标志位为true，则直接返回true，如果标志位为false，则更新为true，并返回false

* clear：清除标志位，将标志位设置为false



它们的逻辑大概如下：



``` C++

#include <iostream>

#include <thread>

#include <atomic>



class my_atomic_flag {

    public:

        my_atomic_flag() {

            val = false;

        }



        bool test_and_set() {

            if(val == false) {

                val = true;

                return false;

            } else {

                return true;

            }

        }



        void clear() {

            val = false;

        }



    private:

        std::atomic<bool> val;

};



my_atomic_flag flag;

int cnt;



void func() {

    while(flag.test_and_set()) {}



    int c = 10000;

    while(c--) {

        ++cnt;

    }



    flag.clear();

}

 

int main(){

    std::thread th1(func);

    std::thread th2(func);



    th1.join();

    th2.join();

    std::cout << cnt << std::endl;



    return 0;

}

```



当test_and_set返回true时，说明标志位已经被设置过，可以理解为锁已经被占用，此时可以等待锁；如果test_and_set返回false时，说明当前线程占用锁，然后就可以执行对应的逻辑，在执行逻辑结束后执行clear清除标志。这种使用方法是典型的自旋锁。



使用atomic_flag还可以实现简单的条件变量，使用test_and_set检查条件是否满足。



### 5  总结



* 原子操作是不能被分割和中断的操作，在多线程并发环境中修改共享数据时，由于数据更新操作不是原子的，会造成数据不一致

* 可以通过加锁操作实现数据的互斥修改，防止数据不一致，但是加锁会降低性能

* C++11提供了std::atomic，它基于底层硬件提供的原子操作能力，为了判断当前环境是否支持原子操作，std::atomic提供了is_lock_free()

* std::atomic对常见的基本数据类型进行包装，并提供了原子级别的数据读取、更新和CAS操作；对于整数类型的原子对象，还支持算术和逻辑运算。std::atomic的主要使用场景就是数据的原子更新和无锁编程

* std::atomic_flag是原子布尔类型，只提供了test_and_set和clear操作，可以实现类似自旋锁、条件变量等能力