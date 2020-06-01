title: C++ 右值引用
date: 2014-12-07 19:50:45
tags: [C++]
---

C++ rvalue reference 是在C++11 中添加的特性。首先看一看它解决的问题:

* 完成了 Move 语义
* 完美转发

首先看一看 `lvalue`和`rvalue`的定义，这个来自于C.

> An lvalueis an expression e that may appear on the left or on the right hand side of an assignment, whereas an rvalue is an expression that can only appear on the right hand side of an assignment

例子:
```cpp
	int a = 42;
	int b = 43;
	// a and b are lvalue
	a = b;
	b = b;
	a = a * b;
	// a * b is an rvalue
	int c = a * b; // ok
```

在C++ 中，前面的方法是一种很直观的方法来分辨`左值`和 `右值`。但是在C++中,上面的定义是不正确的.

>  C++ with its user-defined types has introduced some subtleties regarding modifiability and assignability that cause this definition to be incorrect.

这里，给出另外一种定义:

> An lvalue is an expression that refers to a memory location and allows us to take the address of that memory location via the & operator. An rvalue is an expression that is not an lvalue.

注意到`lvalue`和`rvalue`针对的是`expression`

例子:

```cpp

	// lvalues;
	int i = 42;
	i = 43;
	int *p = &i; // ok i is the lvalue;
	int& foo();
	foo() = 42 ;//ok foo() is an lvalue
	int *p1 = &foo() //ok foo() is an lvalue
	//rvalues;
	int foobar();
	int j = 0;
	j = foobar(); // ok foobar() is an rvalue
	int* p2 = &foobar() ; //error can't take the address of an rvalue;
	j = 42 ;  //42 is an rvalue
```

## 左值和右值的转换

```cpp

int a = 1;     // a is an lvalue
int b = 2;     // b is an lvalue
int c = a + b; // + needs rvalues, so a and b are converted to rvalues
           // and an rvalue is returned
```


a和b都是`lvalue`,在第三行进行了隐式的`lvalue-to-rvalue`的转换，`All lvalues that aren't arrays, functions or of incomplete types can be converted thus to rvalues.` 但是呢，`rvalue`是不能向`lvalue`转换的．但是这不意味着，`右值`不能够产生`左值`的结果.　一元操作符*能够将右值，转成左值．

```cpp
int arr[] = {1,2};
int *p = &arr[0];
*(p+1) = 10; //p + 1 is an rvalue, but *(p + 1) is an lvalue
```

相反，取地址符号(&)能够将左值转成右值

```cpp
int var = 10;
int* bad_addr = &(var + 1); // ERROR: lvalue required as unary '&' operand
int* addr = &var;           // OK: var is an lvalue
&var = 40;                  // ERROR: lvalue required as left operand
```

## Move 语义
假设X是一个拥有一个指针或者是句柄，它们指向了一些资源,比如`m_pResource`，这里的资源指的是任何需要相当大的努力来创建，克隆，和销毁。逻辑上，X类的`copy assignment operator `应该是这样

```cpp
X&X::operator=(X const &rhs)
{
 // [...]
  // Destruct the resource that m_pResource refers to.
  // Make a clone of what rhs.m_pResource refers to, and
  // attach it to m_pResource.
  // [...]
}
```
现在考虑X，这么用:

```cpp
X foo();
X x;
x = foo();
```

最后一行，干了这些事情:

*  析构了x所拥有的资源，
*  复制了从foo() 返回的临时资源
*  析构了临时对象，也就是释放了其拥有的资源

如果交换x和临时对象的资源指针(句柄)，然后用临时对象的析构函数来释放x原有的资源，这样会更高效。换句话说，在某些情况下，我们希望在赋值符号的右边是一个`rvalue`,我们希望`copy assginment` 像这样:

```cpp
//[....]
// swap m_pResource and rhs.m_pResource
// []
```

这就是`move semantics`,我们可以通过`overload copy assginment`来实现这种行为

```cpp
X& X::operator=(<mystery type> rhs)
{
  // [...]
  // swap this->m_pResource and rhs.m_pResource
  // [...]
}
```
既然我们定义了`copy assignment operator`的一个重载函数，我们的`mystery type`必须是一个引用(reference) 。此外，我们需要`mystery type`满足如下的行为：

>  when there is a choice between two overloads where one is an ordinary reference and the other is the mystery type, then rvalues must prefer the mystery type, while lvalues must prefer the ordinary reference.

如果我们现在用`rvalue reference`来代替`mystery type`,你就从本质上看到了`rvalue reference`的定义

## 右值引用

如果`X`是一个类型，那么`X&&`就是一个`Rvalue reference to X`,为了更好的区别,`X&`就被称为`lvaue reference`.`rvalue reference `是一种很像`lvalue reference`的类型，但是呢，有几个例外.当遇到重载函数的解析问题是,`lvalues`倾向于`lvalue reference`,`rvalue` 倾向于`rvalue reference`

```cpp
void foo(X& x);
void foo(X&& x);
X x;
X foobar();
foo(x); //argument is lvalue: call foo(X&)
foo(foobar()); // argument is rvalue: calls foo(X&&);
```

所以，`在编译的时候`，需要考虑实参是`lvalue`和`rvalue`来调用不同的函数。
当然，你可以用`rvalue reference`来重载任何函数，但是呢，在绝大多数情况下，这种类型的重载发生在`copy assignment operator `和`copy constructors`，是为了实现`move semantics`

```cpp
X& X::operator= (const X & rhs);
X& X::operator=(const X&& rhs)
{
   //move semantics: exchange content between this and rhs
    return *this;
}
```

## 一个例子看右值引用

简单的实现一个integer vector

```cpp
class Intvec
{
public:
    explicit Intvec(size_t num = 0)
    : m_size(num), m_data(new int[m_size])
    {
    log("constructor");
    }
    ~Intvec()
    {
    log("destructor");
    if (m_data) {
        delete[] m_data;
        m_data = 0;
    }
    }
    Intvec(const Intvec& other)
    : m_size(other.m_size), m_data(new int[m_size])
    {
    log("copy constructor");
    for (size_t i = 0; i < m_size; ++i)
        m_data[i] = other.m_data[i];
    }
    Intvec& operator=(const Intvec& other)
    {
    log("copy assignment operator");
    Intvec tmp(other);
    std::swap(m_size, tmp.m_size);
    std::swap(m_data, tmp.m_data);
    return *this;
    }
private:
    void log(const char* msg)
    {
    cout << "[" << this << "] " << msg << "\n";
    }
    size_t m_size;
    int* m_data;
};
```
简单的运行以下代码:

```cpp
Intvec v1(20);
Intvec v2;
cout << "assigning lvalue...\n";
v2 = v1;
cout << "ended assigning lvalue...\n";
```
打印的结果如下:
> assigning lvalue…
[0x28fef8] copy assignment operator
[0x28fec8] copy constructor
[0x28fec8] destructor
ended assigning lvalue…

现在我们将一个rvalue赋值给v2

```cpp
cout << "assigning rvalue...\n";
v2 = Intvec(33);
cout << "ended assigning rvalue...\n";
```

结果如下:

```cpp
> assigning rvalue…
[0x28ff08] constructor
[0x28fef8] copy assignment operator
[0x28fec8] copy constructor
[0x28fec8] destructor
[0x28ff08] destructor
ended assigning rvalue…
```

可以明显的看出，多出了一对copy constructor和destructor.这是额外的开销.在c++11中，引入的rvalue reference就可以帮我们实现move semantics，来解除这个开销．

```cpp
Intvec& operator=(Intvec&& other)
{
    log("move assignment operator");
    std::swap(m_size, other.m_size);
    std::swap(m_data, other.m_data);
    return *this;
}
```

结果:
>
assigning rvalue…
[0x28ff08] constructor
[0x28fef8] move assignment operator
[0x28ff08] destructor
ended assigning rvalue…

## 强制Move语义

C++11中，允许对于非右值使用Move语义。一个例子就是`std`中的交换函数：

```cpp
	template<typename T>
	void swap(T&a, T& b)
	{
	    T tmp(a);
	    a = b;
	    b= tmp;
	}
	X a.b;
	swap(a,b);
```

这里没有右值，所以这里的`swap`使用的不是move 语义，在这里我们知道，`Move`语义合适，C++11中，我们可以使用`std::move`将其形参转成`右值`。所以在C++11中，`swap`像这样:

```cpp
template <typename T>
void swap(T& a, T& b)
{
    T tmp = std::move(a);
    a = std::move(b);
    b = std::move(tmp);
}
```
对于实现了Move语义的类型`X`,上面的`swap`提高了效率，对于没有实现`Move语义` 的，其结果与原来类似.
考虑下面简单的赋值表达式

```cpp
a = b;
```

你期待发生了什么?  a中持有的对象将会被ｂ的对象取代，同时，在取代的过程中，a 以前的对象，将会被释放。

```cpp
a = std::move(b);
```

会发生什么?如果 `move`语义（这里是指支持右值引用的Copy Assignment）是通过简单的交换，那么上面的效果就是，a 和 b 持有的对象互相交换。没有任何对象析构。当然，以前ａ持有的对象会随着ｂ的消亡而析构。所以对于简单的交换来实现`Move语义`，我们不知道ａ以前持有的对象什么时候析构。所以我们就陷入了这样的情景:变量ａ被赋值了，但是我们不知道以前ａ持有的对象什么时候析构，它仍然在什么地方存在。当对象的析构没有副作用的时候，这时没有问题的。但是，当析构函数由副作用的时候，问题就出现了.比如
通过析构函数来释放锁,在这种情况下，你需要显示的析构．

```cpp
X& X::operator= (X&& rhs)
{

// Perform a cleanup that takes care of at least those parts of the
// destructor that have side effects. Be sure to leave the object
// in a destructible and assignable state.
// Move semantics: exchange content between this and rhs
   return *this;
}
```

## 右值引用是右值吗?

Ｘ是一个类，其重载了复制构造函数和复制操作符，实现了Move 语义.

```cpp
	void foo(X&& x)
	{
	    X anotherX = x;
	}
```

在函数体中，`哪个`复制构造函数会调用呢？表明上来看，x是一个右值引用，应该调用`X(X&& rhs)`,我们以为声明一个右值引用类型就是一个右值，实际上不是．规则是

* 任何声明名为`右值引用类型`可以是左值，也可以是右值．区分的标准是，如果右值引用类型有名字，那么就是左值，否则是右值.

```cpp
void foo(X&& x)
{
    X anotherX = x; // call X(const X &rhs);
}
X&& goo();
X x = goo(); / / calls X(X&& rhs) 因为右边的部分没有名称．
```

之所以是这样，是因为允许对一个有名字使用move 语义将会变的很诡异，和易出错．`X anotherX = x;` 如果使用 `move`语义，由于x仍然能够被后面的代码引用，事情就变得诡异了．

上面可以总结，如果有名字，其就是一个左值.如果没有名字，那么就是一个右值.

`std::move`表达是返会的右值引用，但是没有名字，所以是一个右值．所以`std::move`的作用是，将形参变成右值，即使其不是．其是通过隐藏名字来实现的．
这里有一个例子:

```cpp
Base(const Base & rhs); // non-move semantics
Base(Base&& rhs); //使用右值引用．
```

如果有一个`Dervied`子类继承`Base`类，如果我们想要`Base`的 move语义的部分能够适用于`Derived`部分．

```cpp
Dervied(Derived && rhs): Base(rhs); //错误,rhs是一个左值．
{
}
Derived(Derived&& rhs):Base(std::move(rhs));//好的．
{

}
```

## Move 语义和编译器优化

```cpp
X foo()
{
    X x;
    // do something to x
    return x;
}
```


考虑到最后一句存在一个值拷贝，你可能会这样修改

```cpp
X foo()
{
    X x;
    return std::move(x);
}
```

这样做会更糟，现在的编译器会做返回值优化．

## 完美转发

右值引用解决的另一个问题是`完美转发`.例子

```cpp
template<typename T,typename Arg>
shared_ptr<T> factory(Arg& arg)
{
    return shared_ptr<T>(new T(arg));
}
```
这里的问题在于是传值调用．

```cpp
template<typename T,typename Arg>
share_ptr<T> factory(Arg& arg)
{
    return shared_ptr<T>(new T(arg));
}
```

很到，但是问题是这个工厂函数不能够在右值上调用

```cpp
factory<X>(hoo());// error if hoo returns  by value;
factory<X>(41);//error
```
通过重载这个函数，来解决上面的问题.

```cpp
template<typename T, typename Arg>
shared_ptr<T>factory(const Arg & arg)
{
    return shared_ptr<T>(new T(arg));
}
```

这种方法的缺点是，如果工厂函数有多个形参，那么每个形参的`const`和`non-const`组合都要提供重载函数，第二个缺点是这种方法阻止了`move`语义的使用．

## 完美转发:解决方案．

引用折叠规则

* A& &变成A&
* A& && 变成A&
* A&& &变成A&
* A&& && 变成A&&

第二，模板推导规则有一种特殊的情况，函数模板的形参为右值引用。

```cpp
template<typename T>
void foo(T&& ):
```

* 当foo在右值上调用的时候，T被解析成A,那么，因此形参变成了`A&&`
* 当foo在左值上调用的时候，那么T被解析成`T&`,因此，根据引用折叠的规则.形参的参数为`A&`.

这时候，我们可以使用右值引用来解决完美转发的问题。

```cpp
shared_ptr<T>factory(Arg&& arg)
template<typename T,typename Arg>
{
    return shared_ptr<T>(new T(std::forward<Arg>(arg)));
}
```

这里`std::forward`的实现.

```cpp
template<typename S>
S&&& std::forward(remove_reference(S&)::type& a) noexcept
{
    return static_cast<S& &&>(a);
}
```
我们定义`A`和`X`是一种类型。假设`factor<A>(x)`被调用在左值类型`X`上,

```cpp
X x;
factory<A>(X);
```


通过特殊的类型推导规则，模板参数Arg被推导成`X&`，那么在`factory` 实例化的过程中，`std::forward` 的相应的为:


```cpp
shared_ptr<A> factory(X& && arg)
{
    return shared_ptr<A>(new A(std::forward<X&>(arg)));
}
X&&& forward(remove_reference<X&>::type& a) noexcept
{
    return static_cast<X& &&>(a);
}
```

最后,类型推导后的函数为

```cpp
shared_ptr<A> factory(X& arg)
{
    return shared_ptr<A>(new A(std::forward<X&>(arg)));
}
X&std::forward(X& a)
{
    return static_cast<X&> (a);
}
```

对于右值的调用:


	X foo();
	factory<A>(foo());

类型推导的过程：注意到`std::forward`的形参`Arg`被解析成`X`

```cpp
shared_ptr<A>factory(X&& arg)
{
    return shared_ptr<A>(new A(std::forward<X>(arg)));
}
X&& forward(X& a) noexcept
{
    return static_cast<X&&>(a);
}
```

工厂函数的形参的类型通过了两个层次的类型推导，传给了`A`的构造函数。两次的类型推导是通过引用。
此外，A的构造函数最后得到也是一个右值引用类型的形参,并且没有名字，所以,这个参数是一个右值。所以A的构造函数在右值上进行了调用. 所以转发保存了移动的语义。

让我们看看`std::move`的实现


```cpp
template<class T>
typename remove_reference<T>::type&&
std::move(T&& a) noexcept
{
  typedef typename remove_reference<T>::type&& RvalRef;
  return static_cast<RvalRef>(a);
}
```

如下代码

```cpp
X x;
std::move(x);
```

根据模板类型推导的规则，那么模板参数`T`会解析成`X&`，最后，std::move最后为:


```cpp
X&& std::move(X&&& a) noexcept
{
    typedef typename remove_reference<X&>::type&& RalRef;
    return static_cast<RvalRef>(a);
}
```

通过`remove_reference<X&>`的解析后


```cpp
X&& std::move(X&&& a) noexcept
{
    return static_cast<X&&>(a);
}
```

所以，对于一个左值，通过在形参中绑定一个左值引用，最后返回一个没有名字的右值引用类型。‘


## 右值引用和异常安全
当你重载复制构造函数和赋值操作符的时候，如果你需要考虑移动语义，那么你应该注意以下规则:

* 对于这些重载函数，你应该不能够抛出异常。因为这些移动语义是非常简单的,可能就是交换指针
* 如果不抛出异常，那么你应该使用`noexcept`关键字。

## 参考资料
http://thbecker.net/articles/rvalue_references/section_01.html



