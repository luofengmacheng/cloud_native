## go语言执行外部排序

### 1 make和new

大家在使用channel时最常用的操作就时使用make()来创建一个管道，而在使用指针时又会使用到new()，那么make()和new()有什么区别呢？

* make()只能用来分配和初始化slice、map、chan，而new()可以分配任意类型
* make()返回的是类型本身，而new()返回的则是指针类型
* 主要的区别就是slice(类比C++中的vector)、map、chan三种类型底层需要复杂的数据结构支持，需要对所支持的数据结构进行初始化，这些类型才能够使用，而new()只是分配对应类型的内存空间，然后初始化为零值，然后返回这块内存空间的地址

### 2 range

在goroutine中处理chan的数据时，range也是个很常用的操作，而且使用方式跟python很类似。同时，针对不同的类型，range返回的数据也有所不同：

如果是对string、slice(数组和切片)进行遍历时，会返回两个元素，第一个元素时索引，第二个元素是值：

``` go
// 输出是：
// 0 2
// 1 3
// 2 4
// 3 5
s := []int{2, 3, 4, 5}
for i, v := range s {
	fmt.Println(i, v)
}
```

如果是对map进行遍历时，会返回两个元素，第一个元素是键，第二个元素是值：

``` go
// 输出是
// a 1
// b 2
// c 3
s := map[string]int{"a": 1, "b": 2, "c": 3}
for i, v := range s {
	fmt.Println(i, v)
}
```

如果是对chan进行遍历时，只会返回一个元素：从chan中获取的元素。

``` go
package main

import "fmt"

func Sort(inputs ...int) <-chan int {
	out := make(chan int, 10)
	go func() {
		for _, v := range inputs {
			out <- v
		}
		close(out)
	}()
	return out
}

func main() {
	p := Sort(1, 2, 3, 4)
	for i := range p {
		fmt.Println(i)
	}
}
```

注意：在for range中有个很经典的坑：

``` go
// 输出的地址是一样的
// 每次遍历时会将p中的元素的内容拷贝到v中，也就是类似值传递，每次访问的不是数组中的元素，而是元素的副本
// 因此，如果想要使用数组或者切片元素的指针，就只能用索引取地址的方式了
p := []int{1, 2, 3, 4, 5}
for _, v := range p {
	fmt.Println(&v)
}
```

### 3 外部排序

排序算法分为内部排序和外部排序，内部排序是数据可以全部加载到内存，操作也直接在内存中进行，外部排序是数据不能全部加载到内存，需要利用硬盘或者网络进行排序。

外部排序最常用的方式就是多路归并排序：

由于内存空间，不能一次性将所有数据加载到内存，需要先将要排序的数据文件进行切分，分别对每个部分进行内部排序，排好序后写入到文件，然后对几个排好序的文件分别读取第一个数据，获取其中的最小元素，即得到所有数据的最小元素，然后写入最终的文件，此时可以将刚才的最小元素所在的排好序的文件的下一个元素读取到内存中，再进行比较，获取到最小值，再写入最终的文件，直到所有的排好序的文件处理完毕。

当然，这里可以做两个优化：

* 归并时每次取得最小的元素时可以用最小堆
* 最终写入文件时可以使用缓存，不需要获取到一个值就写入文件

上面是用C/C++等语言处理外部排序的方法，如果用go语言，则可以充分使用goroutine和chan机制：

当然，整体思路还是归并排序。只是，多个操作之间可以利用chan进行流水线操作。从上面的操作可以看出，整个排序过程可以分成四个部分：对文件进行切分并读取；对每个部分进行内排序；将所有的排好序的部分进行归并排序；将最终的数据写入到文件。每个操作都可以用一个单独的goroutine，然后每个操作之间通过chan进行数据流动。这样的话，整个排序操作就类似于流水线，而且也很容易扩展为分布式结构。

这里我们还是以一个数组为输入构建管道架构的排序：

``` go
// 输入是一个可变长的int型数列
// 依次读取输入数列中的数据，然后写入到管道
// 注意：这里的输出管道需要加上close()操作，否则接收端没办法识别数据是否读取完毕，会产生死锁
func ArraySource(a ...int) <-chan int {
	out := make(chan int, 1024)
	go func() {
		for _, v := range a {
			out <- v
		}
		close(out)
	}()

	return out
}

// 内部排序：对输入管道的数据进行内部排序
func MemSort(in <-chan int) <-chan int {
	out := make(chan int, 1024)
	go func() {
		sli := make([]int, 0)
		for v := range in {
			sli = append(sli, v)
		}
		sort.Ints(sli)

		for _, v := range sli {
			out <- v
		}
		close(out)
	}()
	return out
}

// 归并排序：对若干个输入管道进行二分递归归并
func MergeSort(inputs ...<-chan int) <-chan int {
	if len(inputs) == 1 {
		return inputs[0]
	}
	out := make(chan int, 1024)
	go func() {
		n := len(inputs) / 2

		res1 := MergeSort(inputs[:n]...)
		res2 := MergeSort(inputs[n:]...)
		d1, ok1 := <-res1
		d2, ok2 := <-res2

		for ok1 || ok2 {
			if !ok2 || (ok1 && d1 < d2) {
				out <- d1
				d1, ok1 = <-res1
			} else {
				out <- d2
				d2, ok2 = <-res2
			}
		}

		close(out)
	}()
	return out
}
```