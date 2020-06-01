title: muduo源码分析之mutex
date: 2014-12-05 19:52:18
tags: [C++,Muduo,Multithreading,Source Code Reading]
---

## muduo介绍

这是由[ 陈硕 ]( http://blog.csdn.net/Solstice/  )写的.一种适应性较强的多线程服务器编程模型，下面是作者在开发这个库的时候考虑到的.

> * 线程安全，支持多核多线程
* 不考虑可移植性，不跨平台，只支持 Linux，不支持 Windows。
* 在不增加复杂度的前提下可以支持 FreeBSD/Darwin，方便将来用 Mac 作为开发用机，但不为它做性能优化。也就是说 IO multiplexing 使用 poll 和 epoll。
* 主要支持 x86-64，兼顾 IA32
* 不支持 UDP，只支持 TCP
* 不支持 IPv6，只支持 IPv4
* 不考虑广域网应用，只考虑局域网
* 只支持一种使用模式：non-blocking IO + one event loop per thread，不考虑阻塞 IO
* API 简单易用，只暴露具体类和标准库里的类，不使用 non-trivial templates，也不使用虚函数
* 只满足常用需求的 90%，不面面俱到，必要的时候以 app 来适应 lib
* 只做 library，不做成 framework
* 争取全部代码在 5000 行以内（不含测试）
* 以上条件都满足时，可以考虑搭配 Google Protocol


现在简单的分析来自`Base/Mutex.h`的文件．(注意，后面分析的系列都以v0.1版为基础，主要是要关注实用功能的本身．现在的版本已经是[1.04版]( https://github.com/chenshuo/muduo  ))

## 互次锁的封装

首先来看看源码:

```cpp
class MutexLock : boost::noncopyable
{
     public:
          MutexLock()
          {
            pthread_mutex_init(&mutex_, NULL);
          }

          ~MutexLock()
          {
            pthread_mutex_destroy(&mutex_);
          }

          void lock()
          {
            pthread_mutex_lock(&mutex_);
          }

          void unlock()
          {
            pthread_mutex_unlock(&mutex_);
          }

          pthread_mutex_t* getPthreadMutex() /* non-const */
          {
            return &mutex_;
          }

     private:

          pthread_mutex_t mutex_;
};

class MutexLockGuard : boost::noncopyable
{
     public:
          explicit MutexLockGuard(MutexLock& mutex) : mutex_(mutex)
          {
            mutex_.lock();
          }

          ~MutexLockGuard()
          {
            mutex_.unlock();
          }

         private:

          MutexLock& mutex_;
};

#define MutexLockGuard(x) error "Missing guard object name"
```


首先看看用`RAII`手法封装的互斥锁的`MutexLock`,和`MutexLockGuard`类.`MutexLock`类封装了`mutex`的创建，销毁，加锁，解锁四个操作． 而加锁和解锁的接口不是给用户调用而是给`MutexLockGuard`用的．主要意图:

* 不是要手工调用`lock()`和`unlock()`函数，一切交给栈上的`MutexGuard`对象的构造和析构函数负责，`MutexGuard`对象的生命期等于临界区．这样我们保证 始终在同一函数同一个`scope`里对某个`mutex`加锁和`解锁`,我可以想到用`MutexLock`来封装｀创建｀和`销毁`，但是我没有想到用`MutexGuard`来进一步的保证`加锁`和`解锁`



## 一点解释

首先这个`MutexLock`类 有一个私有变量互斥锁`mutex_`，一些接口:
* MutexLock()　　初始化互斥锁，使用默认的锁的`attribute`.
* ~MutexLock() 释放锁的资源
* locked()　　用于将锁锁住
* unlocked(); //释放锁．

对于互斥锁类(mutex class)我们能要求什么?我想无非就是能够提供对`mutex`能够进行`lock`和`unlock`，当然`muduo`提供的这个类，不只有这样简单的功能，其还提供了判断锁的状态的`isLockedByThread()`和`assertLocked()`(注意这个是在后面的版本提供的，这个版本没有)，整个设计都非常的简单．

注意几点:

*  这个`MutexLock`类继承了`boost::nocopyable`，也就是说`MutexLock`不支持拷贝的语义，也就是说锁不能够复制．从锁真正的意义来看，锁是为了让多个线程能够正确的执行而提出来的，只能有一个线程进入临界区，占有锁，才能够执行临界区的代码．如果一个锁能够拷贝，搞出多个锁来，那么就不是一个线程能够在临界区执行．所以这种锁是部支持复制．
* ` #define MutexLockGuard(x) error "Missing guard object name" `注意到这句，这句是为了防止出现想下面的错误


```cpp
void doit()
{
    MutexLockGuard(mutex);//遗漏了变量名，产生了一个临时变量又马上销毁，结果没有锁住临界区．
    //正确写法MutexLockGuard lock(mutex);
    //临界区
}
```


再看看用RAII手法对MutexLock封装的 MutexGuard

```cpp
class MutexLockGuard : boost::noncopyable
{
    public:
        explicit MutexLockGuard(MutexLock& mutex)
            : mutex_(mutex)
        {
                mutex_.lock();
        }

        ~MutexLockGuard()
        {
            mutex_.unlock();
        }

    private:

        MutexLock& mutex_; //注意这个是引用类型．
};
```


一看就是简洁明了，只有构造函数和析构函数，当然对于这个类是不需要其它的操作的．知道`RAII`手法的同学知道，当我们在获取锁的同时，把它放进类似于`MutexLockGuard`类是为了自己不需要手动管理锁的释放，而且手动释放锁也不妥当，当程序发生异常时，提前退出，使得锁没有及时的释放.


## 几点感受

* 项目的源码风格高度统一,注意到类的成员变量以`_`结尾.
* 函数名，变量名字与其表示功能具有一致性，可读性很好．

