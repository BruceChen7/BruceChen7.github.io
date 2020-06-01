---
layout: post
title: "C++ FAQ Reading之构造函数"
date: 2015-04-01 21:24:31
tag: Cpp
---

* 类的设计应该考虑属于这个类的每个对象的真正属性,这称为不变性.意识是属于这个类真正本质的东西.构造函数的工作就是建立起这种不变性.每个成员函数都应该维护这种不变性.

* 有多种方式来表达多态的语义,可以在运行期间(通过基类对象的指针和引用),也可以是在编译期间

* 分层次的设计类,是为了表达类之间的层次关系,这样是为了简化代码

* coding standards需要具有一致性的philosophy of design and implementations.coding standards是每个organization都需要考虑的.不要自己设置coding standard,即使自己在C++上有相当多的经验.

* xxxx.h和,前者是为了使用C的头文件,C++标准库保证能够使用18个C的头文件,有两种方式使用,一种是xxx.h,一种是cxxx,两种的区别是前者在std和全局名字空间都可以都有声明,或者在std中声明.xxx.h正在被遗弃.当然还有32个使用xxx.h的头文件不与c的头文件对应,比如iostream.h,这些是编译器厂商提供的,它们可能不和标准的版本相同,我们应该注意

* 对应于新的项目,我们应该使用xxxx而不是xxx.h

* 我们最好不用using namespace std,这样会造成改名字下所有的名字都可见.有两种方法来代替这句,一种使用using declaration,一种是对于每个名字都使用std

* 局部变量的声明应该靠近其使用的地方,而不是将所有的局部变量放在一个函数的前面

* 任何一个含有虚函数的类都应该含有虚析构函数

* A class with any of {destructor, copy assignment operator, copy constructor, move assignment operator, move constructor} generally needs all 5

* 一个Fred类的复制构造函数和复制操作符的原型应该是这样的Fred::Fred(const Fred &)和Fred& Fred::operator = (const Fred &)

* 对应于copy assignment,我们应该显示的处理自己复制自己的情况.

* 在初始化对象的成员的时候,应该使用初始化列表initialization lists,而不是赋值.


## 构造函数

下面的`FAQ`来解决构造函数的疑问。

**List x()和 List x的区别 ?**

* 区别很大，前者是一个函数返回List对象，后者是一个List类型的对象`x`

**默认的构造函数都是`Fred::Fred()`吗?**

* 当然不是，不过这是一种典型的形式。默认的构造函数是一个`能够不需要使用实参`就可以调用的构造函数，一种典型的形式是

```cpp
class Fred
{
    Fred(); //默认的构造函数.
};

```

* 另外一种默认的构造函数是需要实参，但是每个形参都有默认的值。

```cpp
class Fred
{
    Fred(int i = 0 ;int j = 1);//这也是默认的构造函数。
};
```

**当创建一个`Fred`对象的数组，哪种构造函数将会被调用?**

```cpp
class Fred
{
    Fred();// 默认的构造函数
};

int main()
{
    Fred a[10]; //将会调用`Fred`的默认的构造函数10次
    Fred*p = new Fred[10]; //同样也会调用Fred默认构造函数10次
}
```


如果一个类没有想上面提供显示默认的构造函数，而是提供了别的默认的构造函数，则上面的代码将会编译不通过。

```cpp
class Fred
{
    Fred(int i,int j); //没有默认的构造函数
};
int main()
{
    Fred a[10]; // error
    Fred *p = new Fred[10]; // error
}
```

然而，即使是一个class拥有默认的构造函数，也`不应该使用数组`(arrays are evil),你应该使用`std::vector`，`vector`可以让你决定使用哪个构造函数，即使不是默认的构造函数.

```cpp
#include <vector>
int main()
{
    std::vector<Fred> a(10,Fred(5,7));
}
```

**构造函数是使用初始化列表，还是赋值操作?**

使用初始化列表，实际上应该把这条当成规则。当然也有例外(例外的情况，我没有理解)使用初始化列表最大的好处就是能够提高性能。比如`Fred::Fred():x_(whatever)`,编译器不需要复制一个单独的`whatever`对象来初始化`x_`.如果使用`Fred::Fred() { x_ = whatever；}`,这种情况将会产生一个临时对象的产生，同时当`；`结束的时候临时对象将会被析构掉。

**使用explicit关键字目的?**

首先，这个关键字是给构造函数和类型转换符用来告诉编译器，某个构造函数和类型转换符号不能够进行隐式的类型转换。下面是没有使用explicit 关键字，比如说:

```cpp
class Foo
{
    Foo(int x);
    operator= (int );
};

class Bar
{
    Bar(double x);
    opearator double();
};

void yourCode()
{
    Foo a = 42;  // Ok,调用Foo::Foo(int)
    Foo b(42);
    Foo d = (Foo)42 ; //Ok,调用Foo::Foo(int)
    int e = d; // Compile-time error
    Bar x = 3.14; //Compile-time error,不能够将3.14转成Bar对象
    Bar w = (Bar)3.14 ;//Ok, 调用Bar::Bar(double x);

}
```

使用`explicit`的关键字后，你将限制这些隐式的类型转换

```cpp
class Foo
{
    public:
        explicit Foo(int x);
        explicit opeartor int();
};

class Bar
{
    public:
        explicit Bar(double x);
        explicit opearator double();//类型转换操作符
};

void yourCode()
{
    Foo a = 42; //compile-time error:不能将42转成Foo类型。
    Foo b(42);  //ok，调用Foo::Foo(int)传递42作为参数
    Foo c = Foo(42); // ok
    Foo d = (Foo)42; //Ok,d调用Foo::Foo(int)

    Bar x = 3.14; //compile-time error:不能够将3.14转成Bar
    Bar w = (Bar)4.14 ;//OK，调用Bar::Bar(dobule)
    double v= w; //编译错误，不能convert w to double
    double u = double(w);// 可以
}
```


**为什么静态的数据成员会在链接的时候报错**


因为静态的数据成员必须在一个编译单元中显示的被定义。如果你没有这么做，就会链接出错。

```cpp
class Fred
{
    public:
     //...
    private:
        static int j_;
};

```
除非你在你的源文件中显示的定义`Fred::j_`，否则则会出现链接错误。像这样:

```cpp
	#include "Fred.h"
	int Fred::j_ = 1;
```

通常定义静态数据成员的地方是`Fred.cpp`

**可以在static const数据成员的声明中加入`= initializer`吗**

可以，但是有限制。这里只能够对整数和枚举类型能够中声明的时候加入。比如：


```cpp
class Fred
{
    public:
        static const int maximum = 42;
        static const std::string t ;
};

//in Fred.cpp
const std::string t = "Hello world";
```

