---
layout: post
title: C++ Managing threads(1)
date: 2014-12-04 16:41:42
tag: C++
---

# Hello world
第一个例子创建一个新的线程．经典的`Hello world`

```cpp
#include <iostream>
#include <thread>
void hello()
{
    std::cout << "Hello Concurrent world\n";

}
int main()
{
    std::thread t(hello);
    t.join();
}
```

`g++ Hello.cpp -o Hello -std=c++11 -pthread`,注意编译这个程序的手法，由于线程库是C++11开始才有的，注意你的编译器能够支持c++11 的标准．

# launching a thread
在上面的例子中，用hello这个函数对象创建了一个thread对象，这个对象的名称叫做`t`,然后就创建了这个线程，这个线程调用了hell函数，`t.join()`表示主线程等待新创建的线程结束。在上面的例子里面只是简单的添加了一个void 任务，没有什么价值，另一种极端的方式是，这个任务，有额外的参数，运行一些独立的操作，而这些操作是通过某种消息系统，线程的停止也是通过信号被告知。用标准库来创建一个`thread`总是归结为创建一个`thread`对象。

<!--more -->
像这样:

```cpp
void do_some_work();
std::thread my_thread(do_time_work);
```

std::thread 也能够和`callable type`工作. 那么什么是`callable type?`

> A callable object is something that can be called like a function, with the syntax object() or object(args);
 that is, a function pointer, or an object of a class type that overloads operator(). The overload of operator() in your class makes it callable.

简单的说是一个重载了`()`的对象，可以像函数那样调用。


```cpp
class background_task
{
    public:
    void operator() () const
    {
        do_something();
        do_something_else();
    }

};
background_task f;
std::thread my_thread(f);
```

在这个例子中，提供的函数对象`function object`被copy 到新建的线程，并且新建的线程开始执行。不过注意一点问题，如果你是用`临时变量`而不是一个`有名字的对象`，那么编译器把其看成一个`函数的声明`，而不是新建一个线程


```
std::thread my_thread(background_task())
```

上面则是声明了一个函数`my_thread()`，它接受一个参数，这个参数是一个没有形参，返回`background_task`的函数指针。my_thread函数返回`thrad`对象。你有两种方式来避免这种情况:


```cpp
std::thread my_thread((background_task()));
std::thread my_thread{background_task()};
```


第一种方式中额外的括号是阻止其成为一个函数的声明，而是将其解释为`std::thread`的变量。第二种方式用`uniform initialization syntax`(不知道这个语法？出门左转 [ uniform initialization and list](http://brucechen.gitcafe.com/C++/syntax/uniform%20Initialization%20and%20Initializer%20Lists.html ))将其声明为变量。

还可以使用`lambada expression`来避免这个问题:

```cpp
std::thread my_thread( [] {
    do_something();
    do_something_else();
 });
```

一旦你创建了一个线程，你就必须决定是要在主线程中等待其结束，还是`分离(detach)这个线程` 。注意这个`决定必须是你在创建的线程被destroyed之前`。如果你不想等待你创建的线程结束，那么你就必须保证创建的线程访问的数据都有效。很常见的一种用情况就是创建的线程访问`共享的变量`，而主线程已经结束了。


```cpp;
struct func
{
    int &i;
    func (int& i_):i(i_){}
    void operator()()
    {
        for( j = 0 ; j < 1000 ;j++)
        {
            do_something(i);
        }
    }
};
void oops()
{
    int some_local_status = 0;
    func my_func(some_local_state);
    std::thread my_thread(my_func);
    my_thread.detach();
}
```

这个例子在`oop`这个函数运行的时候是没有任何问题的，但是由于通过调用`my_thread.detach()`声明你不等待`ｍy_thread()`线程的结束，所以当`oop`这个主线程退出的时候，`do_something`这个函数就会访问一个已经被销毁的对象。就像单线程一样，在函数退出的时候访问一个局部的引用和指针不是一个好主意。
解决这种问题的一个方法就是让创建的线程拥有这个数据，而不是与其它的线程共享这个数据。

# waiting  for a thread  to complete

如果你需要等待新建的线程完成，你必须调用`.join()`方法，通过这个方法，在上面的问题中，主线程能够保证新建的线程能够在主线程结束之前退出。在这种情况下，通过创建一个新线程来调用函数是没有什么意义的，因为主线程什么都没有做，就在等待新建的线程。这个`.join()`方法是清理线程的内存。所以清理结束后，就不存在这个线程了。


# waiting in exceptional circumstances

在前面提过，你必须在线程被`destroyed` 之前调用`join()`或者是`detach()`,一般来说，创建线程，然后`detach()`是没有任何问题的，但是你如果想要等待一个线程结束，你就必须考虑`join()`的位置，也就是说当线程开始执行，但是在主线程调用`join()`之前抛出异常，那么调用`.join()`就可能被跳过。下面是解决这个问题的一种方式:

```cpp
struct func;
void f()
{
    int some_local_state = 0;
    func my_func(some_local_state);
    std::thread t(my_func);
    try
    {
        do_something_in_current_thread();
    }
    catch(...)
    {
        t.join();
        throw;
    }
    t.join();
}
```

这种方法显得很冗长，还有没有更好的方法:注意到我们的问题是什么？就是不管有没有抛出异常，我们必须创建的线程内存能够被回收？你会想到什么方法？是不是经典的RAII手法呢？(不清楚RAII的，出门左转,[RAII](http://brucechen.gitcafe.com/C++/Advanced/RAII.html)  )

```
class thread_guard
{
    std::thread &t;
    public:
        explicit thread_guard(std::thread& t_):t(t_){}
        ~thread_guard()
        {
            if(t.joinable())
            {
                t.join();
            }
        }
        // they are not automatically provided by the compiler;
        thread_guard(thread_guard const& ) = delete;
        // the same as the above
        thread_guard& operator= (thread_guard const &)=delete;
  };
  struct func;
  void f()
  {
        int some_local_function  = 0;
        func my_func(some_local_func);
        std::thread t(my_func);
        thread_guard(t);
        do_some_thing_in_current_thread();

   }
};
```



如果你不需要等待一个线程完成，那么你可以通过`detach`这个线程来避免这个异常安全问题。这样的话，创建的新线程和`std::thread`对象之间的关系也就撇开了。即使是当`std::thread`对象被destroyed，也不会调用`std::terminate() `。为了能够调用`detach()`的方法，你必须保证这个线程对象关联了一个能够执行的线程，这个能够调用`join()`的要求是一样的。`You can only call t.detach for a std::thread object t when t.joinable();`

```cpp
std::thread t(do_background_task);
t.detach();
assert(!t.joinable());
```

# Running threads in the background

调用`detach()`方法能够让线程在后台运行，没有直接的方法来与之通信。没有方法等待其`terminate`，没有办法获得`std::thread`对象来`reference`它。所以，线程的拥有权利和控制权利交给了`C++ Runtime library`，它来保证资源的｀reclaim｀回收。
detached thread 通常被称为`daemon thread`，也就是说它们运行在后台，没有显示的用户接口。这些线程通常是长期运行的。它们在应用程序的整个周期都会运行，运行一些后台的任务，比如说监视文件系统，清楚一些没有用的缓存。
