---
layout: post
title: C++ Manage Thread(2)
date: 2014-12-09 22:02:52
tags: Cpp
toc: true
---

# Passing arguments to a thread function
接着上面的讲，将形参传递给一个线程函数是非常简单的．但是要注意，默认的情况下被传递的形参被`copy` 到线程内的存储区，这样新建的线程就能够访问这个内存区．即使是函数所要求的是一个`reference` ,也就是先`copy到新建的线程内部的内存`，然后在线程中，的引用的就是`指向内部的数据`．这里有一个例子:

```cpp
void f(int i,std::string const &s);
std::thread t(f,3,"hello");
```

<!--more-->
新创建的线程用ｔ关联,线程将调用`f(3,"hello")`,注意到第二个参数是`string `类型，而字符串"hello"，传给ｆ时，是`const char *` ,`当前仅当` 在`新的线程`中会转成`string`类型．注意下面的例子:

```cpp
void f(int i,std::string const &s);
void oops(int some_param)
{
    char buffer[1024];
    sprintf(buffer,"%i",somp_param);
    thread t(f,3,buffer);
    t.detach();

}
```

在这个例子中，有指针指向了`local variable`,也就是`buffer`，当可能在buffer转成string的时候，oops可能已经退出了，这就导致了未定义的行为．解决的方法就是在传给`thread`构造函数的之前，转成string对象．

```cpp
void f(int i,std string const &s);
void oops(int some_para)
{
    char buffer[1024];
    sprintf(buffer,"%i",some_para);
    std::thread t(f,3,std::string(buf));
    t.detach();
}
```

这个例子中，问题在于隐式的类型转换，要将指向`buffer`指针转成string类型，因为`std::thread`的默认构造函数复制了提供的`value`,而没有类型转换．
当然，有可能这是你不需要这样的行为．默认是`对象被复制`，而你需要的是一个`引用`．

```cpp
void update_data_for_widget(widget_id w,widget_data& data);
void oops_again(widget_id w)
{
    widget_data data;
    std::thread t(updata_data_for_widget,w,data);
    display_status();
    t.join();
    process_widget_data(data);
}
```

尽管`updata_data_for_widget()`期望的第二个参数是`引用`来传递，但是`std::thread()`的默认构造函数则是`复制`了给的实参，而不是`reference to  the data`，所以结果是引用的是线程内部的`data`,结果是，当线程结束，内部的数据被清除，所以被传递的`data`并没有改变．解决的办法如下:


```cpp
std::thread t(updata_data_for_widget,w,std::ref(data));
```


这样传递的就是`the reference to the data`．而不是`the reference to  a copy of  data`

# Transfering ownership of a thread
在一些场景中，你可以创建一个函数，这个函数创建了一个`background thread`，但是你想把线程的`ownership`给调用函数，而不是等待其完成．或者想法，你想创建一个线程，然后将线程的所有权交给等待其完成的函数．

这就是在`std::thread`中`move`支持的来源．正如在`C++ Standard Library`中的`std::ifstream`和`std::unique_ptr`是`movable`而不是`copyable`，thread也是其中之一．

```cpp
void some_function();
void some_other_function();
std::thread  t1(some_function);
std::thread t2=std::move(t1);
t1 = std::thread(some_other_function);
std::thread t3;
t3 = std::move(t2);
t1 = std::move(t3);
```


看上面的代码，在所有的这些所有权转移之后，t1关联了运行`some_other_function`的线程，t2没有关联线程，t3关联了`some_function`线程．在上面的例子中，我们发现t1首先是关联了`some_function`这个线程，然后关联了`som_other_function`，最后又关联了`some_function`,在已经关联了线程的情况下，你不能够简单的用另一个线程对象来`move`.这种情况下`std::terminate()`会调用用来终止程序，你不能够简单的`drop a thread  by assgining a new value to the std::thread object that manages it`.真正的`thransfer ownership of a thread`方式如下:

```cpp
void f()
{
	std::vector<std::thread> threads;

	for(unsigned i = 0 ; i < 20 ; ++i)
	{
		threads.push_back(std::thread(do_work,i));
	}

	std::for_each(threads.begin(),threads.end(),std::mem_fn(&std::thread::join()));

}

```


# Choosing the number of threads at runtime

`std::hardware_concurrency()`给出了给定程序能够真正并发的线程数，在多核的系统中，可能是`cpu`核数．注意到这只是个`hint`，如果没有这个信息，则这个函数返回０.

```cpp
#include <vector>
#include <thread>
#include <iostream>
int main()
{
    std::cout<<std::thread::hardware_concurrency(); // return 2
    return 0;
}
```




这里给出实现并行加法的一个实现，包含了前面讲到了细节.

```cpp
#include <vector>
#include <thread>
#include <iostream>
#include <numeric>
#include <algorithm>

template <typename Iterator,typename T>
struct accumulate_block
{
    void operator()(Iterator first, Iterator last, T& result)
    {
        result = std::accumulate(first,last,result);
    }
};

template <typename Iterator,typename T>

T parallel_accumulate(Iterator first,Iterator last, T init)
{
    unsigned long const length = std::distance(first,last);

    //length is 0 return the init value
    if(!length)
        return init;

    unsigned long const min_per_thread = 25;
    unsigned long const max_threads = (length + min_per_thread -1)/min_per_thread;

    //hardware_threads
    unsigned long const hardware_threads = std::thread::hardware_concurrency();

    unsigned long const num_threads = std::min(hardware_threads != 0 ? hardware_threads:2,max_threads);

    //the number of entries for each  thread to process
    unsigned long const block_size = length/num_threads;


    // to store the intermediate results
    std::vector<T> results(num_threads);

    //you have already one thread which is the main thread
    //so the size of threads is num_threads  - 1;
    std::vector<std::thread> threads(num_threads - 1);

    Iterator block_start = first;

    for(unsigned int i = 0 ; i < (num_threads -1); ++i)
    {
        Iterator block_end  = block_start;
        std::advance(block_end,block_size);

        threads[i] = std::thread(
                                    accumulate_block<Iterator,T>(),
                                    block_start,
                                    block_end,
                                    std::ref(results[i])
                                );
            block_start = block_end;
    }

    accumulate_block<Iterator,T>() (

                block_start,last,results[num_threads-1]);


    std::for_each(threads.begin(),threads.end(),std::mem_fn(&std::thread::join));

    return std::accumulate(results.begin(),results.end(),init);
}


int main()
{
    std::vector<int> vec{1,2,3,4,5};
    std::cout << parallel_accumulate(vec.begin(),vec.end(),10);

    return 0;

}
```



# identifying threads

thread 标识符的类型为`std::thread::id`,一个线程可以通过它关联的`thread`对象,调用`get_id()`来获取．如果一个线程没有关联的`thread`对象，调用`get_id`将返回默认`std::thread::id`对象，表示`not any thread`，当前线程的`id`可以通过`std::this_thread::get_id()`来获取．

* `std::thread::id`类型是可以比较和复制的，如果标识符相等，则表示是同一个线程，或者是都是`not any thread`，如果不相等，表示不同的线程，或者是一个是线程，另一个线程`no any thread`
* 　STL 提供了对`std::thread::id`足够多的比较操作，它们能够被用作`关联容器`的`键值`,或者是`排序`等任何合适的比较方式．
* 输出`std::thread::id`,你可以这样做`cout << std::this_thread::get_id();`,输出的结果与实现有关．



