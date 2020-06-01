title: C++11中的类型推导
date: 2014-12-13 21:33:47
tags: [C++]
---


## 理解模板类型推导


先看看一小段伪代码,这里描述的是一个函数模板

```cpp
template <typename T>
void f(ParamType param);
```



函数的调用可能像这样`f(expr)`，在编译的时候，编译器 利用`expr`推导出了两个类型，一个是T,另一个是整个`形参类型`，但是整个形参类型，通常来说是有一些修饰符号的,比如说`const`，和引用`&`,比如模板像这样声明:

```cpp
template <typename T>
void f(const T& )
```

当我们这样调用的时候:

```cpp
int x = 0
f(x);
```

`T`就被推倒成`int`，`ParamType`就被推倒成`const int &`;


我们很希望T能够被推倒成被传递的形参类型。正如上面所看到的一样，`x`是`int`，T被推倒成了`int`,但是并不总是这样的。对`T`的类型`推倒`不仅仅更`expr`(理解为实参)有关，而且和整个形参的类型(加上修饰符)有关。有三种情况:


* 整个形参是是指针和reference，但不是通用引用
* 整个形参是`通用引用`
* 整个既不是指针也不是引用


后面的讨论都是以下面的伪代码来说的:

```cpp
template<typename T>
void f(ParaType param);
f(expr);
```

## 形参是一个引用类型或者是指针，但不是通用引用

这是最简单的一种情况,类型推到的规则如下:

* 如果实参的类型是引用，则忽视其`引用`的性质，但是其`constness`不会被忽视
* 然后将实参的类型推倒成形参`T`的类型。

这里有一个例子:


```cpp
template<typename T>
void f(T& param); //param is a reference
```

我们有下面的声明:


```cpp
int x = 27;  //x is an int
const int cx = x; //cx is a const int
const int& rx = x; // rx is a reference to x as a const int
```


下面是类型推导的结果:


```cpp
f(x); //T is int, param's type is int&
f(cx); // T is const int,param's type is const int&
f(rx); //T is const int ,param's type is const int&
```


在第二个调用和第三个调用中，注意到`cx`和`rx`都是`const value`, T 就被推导成`const int`.所以通过传递`const object`给一个`T&`模板形参是安全的，因为`T`也推导成`const`

注意到第三个例子,即使`rx`的类型是引用，但是呢T被推导成非引用,这是因为实参rx的引用特性在类型推导中被忽略掉

当我们将`f`的形参从`T&`,改成`const T&`时，the `constness`of `cx`和`rx`继续保留了下来，但是呢由于param(形参)是一个`reference to const`，所以在类型推导中,`const`不再是T的一部分

```cpp
template<typename T>
void f(const T& param);
int x = 27;
const int cx = x;
const int& rx = x;
f(x); //T is int ,param's type is const int&
f(cx); // T is int ,param's type is const int&
f(rx); // T is int ,param's type is const int&
```

正和上面的类型是一样的，在做类型推导的时候，实参的`reference`是被丢弃的。如果是形参是指针，也是一样的

```cpp
template<typename T>
void f(T* param);
int x = 27;
const int *px = &x;
f(x); // T is int ,param's type is int *
f(px); // T is const int param's type is const int *
```


##形参是通用的引用

当模板接受`通用引用`作为自己的形参的时候,类型推导就不这样明显了.这里的形参被声明为｀右值引用`.这里有两条规则:

* 当`expr`是一个左值的时候,`T`和`paramtype`都被推到成左值.这里有两点要注意:1.这是唯一的情况,T能够被推导成为引用.2,尽管形参声明为右值引用,但是却被推导成左值.
* 如果`expr`是一个右值,使用`case 1`的规则.

例子:

```cpp
template<typename T>
void f(T&& param);
int x = 27;
const int cx  = x;
const int& rx = x;

f(x); // x is a lvalue , so T is int& ,paramtype is int &
f(cx); // cx is a lvalue ,so T is const int&, paramtype is const int &
f(rx); // rx is a  rvalue,so T is const int&  paramtype is const int&
f(27); // 27 is rvalue,so T is int param type is int &&
```




## 形参既不是指针和引用

```cpp
template <typename T>
void f(T param)
```

这就意味着`param`是实参的`copy`,在类型推导时：

* 如果`expr`的类型是`reference`，丢弃`reference`部分
* 如果`expr`的类型含有`const`，也丢弃，`volatile`也丢弃．


```cpp
int x = 27;
const int cx = x;
const int& rx = x;
f(x); // T's and param's types are both int
f(cx); // T's and param's types are both int
f(rx); // T's and param's types are both int
```

由于是传值，所以实参和形参完全是两个不同的对象，所以`const`推导的时候被丢弃．实参不能够改变，不意味着形参不能够改变．注意到`const`被忽略掉，只有在`传值`引用的时候被忽略，在其它时候，`const`被保留时．


