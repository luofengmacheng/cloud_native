## c++的lambda表达式

### 1 lambda表达式

lambda表达式可以理解为匿名函数，也就是没有名字的函数，既然是函数，那也遵循函数使用的一些规则，例如，传参方式等。

lambda表达式的一般形式是：

`[捕捉列表] (参数列表) mutable -> 返回值类型 { 函数体 }`

捕捉列表：跟参数列表有些类似，可以用于访问外部的变量
参数列表：函数的参数，支持使用值传递和引用传递，如果没有参数，可以省略
mutable：默认情况下，通过`值传递的捕捉列表参数`是只读的，也就是不允许进行修改，类似于const，可以通过加上mutable关键字，表明在表达式中会对该变量进行修改
返回值类型：通常不用写，让编译器直接推导返回值类型
函数体：使用捕捉列表和参数列表进行计算，得到返回值

``` C++
[] {
	cout << "hell" << endl;
}
```

上面就是一个最简单的lambda表达式，lambda表达式从行为和形式上都像函数，而且调用方式也很像。

``` C++
auto func = [] {
	cout << "hell" << endl;
};

func();
```

将lambda表达式给到变量func，然后就可以将它当做一个函数执行。

### 2 捕捉列表 vs 参数列表

捕捉列表和参数列表都可以传递外部的变量，并且都支持值传递和引用传递，例如：

``` C++
#include <iostream>
using namespace std;

int main() {
	int a = 1;
	int b = 2;

	auto l = [&a](int &b) {
		++a;
		++b;
	};
	l(b);
	cout << "a=" << a << endl;
	cout << "b=" << b << endl;

	return 0;
}
```

采用引用传递分别给到捕捉列表和参数列表，然后在lambda表达式内部修改变量的值，最后在lambda表达式外部打印变量的值，可以看到采用捕捉列表和参数列表达到了相同的效果，都对环境中的变量进行了修改。当然，也会发现两者的不同：使用捕捉列表时，在定义时指定变量，在调用时就不用指定；使用参数列表，在定义时只是指定类型，在调用时需要传参。

因此，捕捉列表相当于在对象构造函数阶段将外部的变量放到了对象的内部，作为初始化的一种手段；而参数列表相当于是在调用对象的operator()方法时给的参数。两者赋值的时机有所不同，而且，对于捕捉列表来说，在定义lambda表达式时就知道用的变量是什么，而参数列表是需要在调用传递参数时才知道用的变量是什么。

### 3 lambda表达式的传递

使用lambda表达式当然不是为了作为函数调用，普通的函数和内联函数可以满足需求，使用lambda表达式通常是为了将lambda表达式作为条件或者执行逻辑传递出去。

#### 3.1 函数作为形参

``` C++
#include <iostream>
using namespace std;

typedef void (*print)(std::string);

void func(print pfunc) {
        pfunc("hello");
}

int main() {
        int a = 1;
        int b = 2;

        auto l = [](std::string msg) {
                cout << msg << endl;
        };
        func(l);

        return 0;
}
```

首先定义print类型的函数指针，然后定义了func函数，它接受print类型的参数，然后调用它，而在主函数中，定义了接受相同形参的lambda表达式，然后传给func函数。

上面的函数指针定义在C中比较常见，而在C++中通常会使用std::function：

``` C++
#include <iostream>
#include <functional>
using namespace std;

void func(std::function<void(std::string)> pfunc) {
        pfunc("luofeng");
}

int main() {
        int a = 1;
        int b = 2;

        auto l = [](std::string msg) {
                cout << msg << endl;
        };
        func(l);

        return 0;
}
```

在定义函数时形参直接通过std::function给出，而不用额外定义函数指针，调用方式跟之前的函数指针一样。

上面就是将lambda当做函数传递的示例，在实际使用中，比较常见的有下面两种场景。

#### 3.2 场景1：条件表达式

``` C++
#include <iostream>
#include <set>
using namespace std;

int main() {
	auto cmp = [](int lh, int rh) {
		return lh > rh;
	};

	set<int, decltype(cmp)> my_set(cmp);
	my_set.emplace(2);
	my_set.emplace(5);

	for(const auto& element: my_set) {
		cout << element << " ";
	}

	return 0;
}
```

std::set的第二个模板参数是一个比较函数，在这里先定义了一个lambda表达式cmp，然后用cmp来定义my_set，然后向set中插入元素，最后再遍历，这里有几个点需要注意：

* std::set的第二个模板参数是个函数指针，也就是函数类型，而lambda表达式是个对象，因此，需要用decltype获取lambda表达式的类型，因为lambda表达式的类型是编译器自己生成的，为了能够在代码中方便获取类型，C++11提供了decltype获取lambda的类型
* std::set的构造函数需要指定实际的比较函数，因此，这里传递lambda变量
* 向std::set中插入元素使用的是emplace，而不是通常的insert，两者的区别在于，使用insert时，会先构造一个临时对象，然后调用临时对象的拷贝构造函数在std::set中创建对象，而使用emplace时，可以直接使用参数在std::set中创建对象，减少一次拷贝构造函数的调用，当然，这里的类型是int，用insert和emplace就没啥区别

#### 3.3 场景2：线程的运行表达式

C++11中引入了std::thread在语言级别支持线程：

``` C++
#include <iostream>
#include <thread>
using namespace std;

int main() {
        thread t([](int loop) {
                        while(--loop) {
                            cout << this_thread::get_id() << " " <<  loop << endl;
                        };
                        }, 10);

        t.join();
}
```

首先使用thread创建一个线程，线程的构造函数接受两个参数，一个是函数，另一个是参数列表，这里用lambda表达式提供函数，lambda表达式有一个整型参数，然后将参数列表设置为10，作为lambda表达式的参数，最后调用thread的join等待线程执行结束。

这里还使用了std::this_thread这个ns中的get_id()来获取线程ID，在std::this_thread中有4个函数：

* get_id()：获取线程ID
* yield()：让出CPU
* sleep_until()：阻塞当前线程，休眠直到某个时间点
* sleep_for()：阻塞当前线程，休眠一段时间

而std::thread类提供的函数主要有以下几个：

* get_id()：获取线程ID
* join()：启动线程，并等待线程执行结束
* detach()：线程就成为后台的守护线程
