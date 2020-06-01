---
layout: post
title: effective cpp笔记
date: 2019-07-31 19:04:04
tag: Cpp
post_list: "date"
home_btn: true
toc_level: 4
footer: true
---

## const 与 #define的区别

* 编译器处理方式不同：define宏是在预处理阶段展开，const常量是要在编译运行阶段使用
* 类型和安全检查不同：define宏没有类型，不做任何类型检查，仅仅是展开。const常量有具体的类型，在编译阶段会执行类型检查
* 存储方式不同：define宏仅仅是展开，有多少地方使用，就展开多少次，不会分配内存。const常量会在内存中分配(可以是堆中也可以是栈中。

## 尽可能的使用const

* 将某些东西声明为const，可以帮助编译器检查错误。const可以被施加到任何作用域的对象，函数参数，函数返回类型，成员函数本体。
* 编译器强制**bitwise constness**，但是我们写程序的时候，应该使用概念上的常量性。
* 当const和non-const成员有着等价的实现，另non-const的版本调用const版本，来避免代码重复。

const的语义之一就是告诉编译器和其他程序员**某值应该不变**，只要这个是**事实**，你就确实**应该说出来**。关键字**const多才多艺**，它能够修饰**全局作用域的常量**，修饰文件，函数，块作用域中被生命为**static的对象**，你可以用它修饰class内部的static和non-static的成员变量，你也可以使用指针自身，指针所指物，或两者都是const。

### const指针

```cpp
char greeting[] = "hello";
char *p = greeting;
const char *p = greeting;
char* const p = greeting;  // const pointer, non-const data
const char * const p = greeting;
```
**指向非常量的指针**可以**转化成指向常量的指针**，而**指向常量的指针转成非常量的指针则是非法的**。

### const迭代器

迭代器以指针来塑模出来的，迭代器的作用就像个T* 的指针。声明迭代器为const就像声明指针为const一样，表示该迭代器不得指向不同的东西，但是它所指向的值可以改动。如果你需要表明，你的迭代器不能够改变其所指向的东西，那么你应该使用const_iterator

### const修饰函数的返回值
令函数返回一个常量值，往往能够降低应客户造成的意外，又不至于放弃安全性和高效性。

```c
class Rational { ... };
const Rational operator*(const Rational& lhs, const Rational& rhs) ;
```
可能你会说，为什什么会返回const对象？原因可能误用成如下:

```cpp
Rational a, b;
(a * b) = c;
if( a * b = c) {
}
```
### const成员函数

const修饰函数，目的确定该成员对象可用于**const对象身上**。两个成员函数如果只是常量性不同，**可以被重载**。在**类X非常量成员函数**，this指针是**X \*const**类型，因为其是一个**常量指针**，而其指向的对象是一个**非常量**，所以该对象是**可以修改的**。而在类X的常量成员中，this的指针类型是**const X* const**，所以其指向的对象为一个**常量**，所以其是**不能够修改的**。

那么什么样的成员函数具有const性质了?  一种认为const成员函数不能够改变类的任意一个bit，就可以。这就是C++对常量性的定义，因此const成员函数不可以改变对象内任何**non-static**的成员变量。

不幸的是，虽然很多成员函数能够通过bitwise 测试，更确切的说是，一个更改了指针所值之物的成员函数，虽然不能够算const，但是如果只有指针隶属于对象，那么称此函数为**bitwise const** 不会引发 编译器异常。这会导致反直观的结果。

```cpp
class CTextBlock {
    public:
    // bitwise const 声明
    char& operator[](std::size_t position) const  {
        return pText[position];
    }
    private:
        char* pText;
};
```
虽然operator[]的实现不改变pText，但是它是bitwise const，所有编译器都这么认定，但是，你看会发现什么事情？

```cpp
const CTextBlock cctb("Hello");
char *rc = &cctb[0];
*pc = 'J' ; //现在cctb 拥有的Jello的内容
```
创建了一个常量对象并设置某值，而且对它至调用了const成员函数，但是还是改变了其值。所以就导出了logical constness。logical constness 认为一个const成员函数可以修改它所处理的对象内的某些bits，但只有在可客户端检查不到才如此。

```cpp
class CTextBlock {
    public:
        ...
        std::size_t length() const;
    private:
        char* pText;
        std::size_t textLength;
        bool lengthIsValid;
};
std::size_t CTextBlock::length() const {
    if( !lengthIsValid) {
        textLength = std::strlen(pText);  //错误，在const成员函数内不能够修改textLength及lengthIsValid
        lengthIsValid = true;
    }
}
```

由于编译器坚持**bitwise constness**，所以上述的代码编译通不过，那么要坚持**logical constness** ，怎么修改，使编译器通过了？解决办法就是利用mutable 释放掉 non-static 成员变量的bitwise constness 约束。

```cpp
class CTextBlock {
    public:
        ....
    private:
        char* pText;
        mutable std::size_t textLength;  // 这些成员函数总是可以被修改，即使在const成员函数中。
        mutable bool lengthIsValid; //
};
```
类的非静态成员可以声明为**mutable**，这样它们的值可以被该类的常量成员函数（包含非常量成员函数）修改。

## 确定对象被使用前先被初始化

### 要点

* 为内置对象进行手工初始化，因为C++不保证初始化它们
* 构造函数最好使用初始化成员列表，而不要在构造函数中，使用赋值操作。初始值列列出的成员变量，其排列次序应该和它们在class中声明的次序相同。
* 为了免除跨编译单元之初始化次序问题，请使用local static 对象替换non-local  static 对象。

C++ 有着十分固定的”成员初始化次序”。次序总是相同： base class早于其derived classes 被初始化，而class 的成员变量总是以其声明次序被初始化。

C++ 对”定义于不同编译单元内的non-local static 对象”的初始化次序并无明确定义。为免除”跨编译单元之初始化次序”问题，请以local static 对象替换non-local static 对象。

所谓static对象，就是其寿命从被构造函数构造出来，直到程序结束为止。因此，stack和heap-based的对象，全都被排除，这种对象包括全局对象，定义于namespace作用域的对象，在class内，在函数内，以及在file作用域中被声明为static的对象，函数内的static对象，被称为local-static对象，其它的static对象为non-static对象。程序结束时，static对象会在main()函数结束时析构。

所谓**编译单元**是指**产生单一的目标文件的源文件**，基本上**它是单一的源代码文件加上其包含的头文件**。现在我们关系的的问题至少设计2个源码文件，每一个至少内含一个non-local static 对象，也就是该对象是global或位于namespace作用域中，抑或是class内或file作用域中声明为static。

```cpp
class FileSystem { ... };
 // 代替tfs对象
FileSystem& tfs()  {
  static FileSystem fs; // 以local static的方式定义和初始化object
  return fs; // 返回一个引用
}
class Directory { ... };
Directory::Directory( params )  {
  ...

  std::size_t disks = tfs().numDisks();
  ...
 }

Directory& tempDir() // 代替tempDir对象，{
  static Directory td;
  return td;
}
```

## 了解C++默认编写并调用了哪些函数

### 要点

* 了解编译器暗自为class创建了**默认构造函数**，**赋值构造函数**，**复制赋值操作符**，以及**析构函数**。

```cpp
class Empty {
public:
    Empty() { ... }
    Empty(const Empty& rhs) { ... )
    ~Empty( ) { ... }
    Empty& operator=(const Empty& rhs) { ... }
 };
```
需要注意的是，只要你显式地定义了一个构造函数（不管是有参构造函数还是无参构造函数），编译器将不再为你创建default构造函数。


## 若不想使用编译器自动生成的函数，就该明确的拒绝

为驳回编译器自动提供的机能，可能将相应的成员函数声明为private并且不予以实现。使用Uncopyable这样的base class也是可以的。


## 为多态基类声明**virtual析构函数**

### 要点

* 多态性质的基类应该声明一个virtual析构函数。如果类带有任何virtual函数，它就应该拥有virtual析构函数。
* 类的设计目的如果不是作为基类使用，或不是为了具备多态性，就不该声明virtual析构函数

设计一个计时类，如下:

```cpp
class TimeKeeper {
public:
    TimeKeeper();
    ~TimeKeeper();
};
class AtomicClock: public TimeKeepr {....};
class WaterClock: public TimeKeepr{.....};
```
如果你使用一个工厂，来创建不同的计时方法：

```cpp
TimeKeeper* getTimeKeeper();
```
这样，getTimeKeeper()返回的对象必须位于heap。因此为了避免资源泄漏，必须对每个对象适当的delete。

```cpp
TimeKeep* ptk = getTimeKeeper();
...
delete ptk;
```
问题在于getTimeKeeper 返回的指针指向的是一个子类的对象的是，通过基类指针来delete被删除时，而基类是没有带virtual的析构函数的时候，会引发灾难，因为子类的成分没有销毁，子类的析构函数没有调用。这样的问题，可以通过给TimeKeeper添加一个virtual析构函数来解决。

无端的对将每一个类的析构函数声明为virtual函数也是错误的，只有在类中含所有至少一个virtual函数的时候才声明析构函数为virtual。

## 别让异常逃离析构函数
* 析构函数**绝对不能够吐出异常**，如果被析构函数调用的函数可能抛出异常，析构函数**应该能够捕捉任何异常，然后吞下异常或结束程序**。
* 如果客户需要对某个操作函数运行期间抛出的异常做出反应，那么class应该提供一个普通函数（而非析构函数）执行该操作。

C++**并不禁止析构函数吐出异常**，但是不建议你这样做。

```cpp
class Widget {
    public:
        ...
    ~Widget() {
    }
};
void doSomething() {
    std::vector<Widget> v;
}
```

当vector v 销毁的时候，它有责任销毁所有的Widgets，如果v内含了10个Widgets，如果析构第一个的时候，有个异常被抛出。其他九个Widgets还是应该被销毁，但是如果第二个Widgets也抛出了异常。在两个异常同时存在的条件下，程序如果不结束执行，就是会导致未定义的行为。及时不在Vector中，只要析构函数抛出异常，程序也会过早的退出或出现未定义的行为。下面是一个数据库的资源管理类：

```cpp
class DBConn　{
public:
    ...
    ~DBConn()  {
        db.close();
    }

};
```
在析构函数中，关闭数据库的连接，无可厚非，但是如果**该调用导致了异常**，就会**传播该异常**，就是**允许异常离开析构函数**。那就会造成问题。如何解决呢？

```cpp
DBConn::~DBConn() {
    try {
        db.close();
    } catch(...) {
        //...
    }
}
```

## 绝对**不要在构造函数和析构函数中调用virtual函数**

### 要点

* 在**构造函数和析构函数不要调用virtual函数**，因为**这类调用从不降至子类**。

## 复制对象时勿忘其每一个成分

本条款阐释了复制对象时容易犯的一些错误，给出的教训是：

* Copying 函数应该确保复制**对象内的所有成员变量**及**所有base class 成分**。
* 不要尝试以某个copying 函数实现另一个copying 函数。应该将共同机能放进第三个函数中，并由两个coping 函数共同调用。换句话说，如果你发现你的copy 构造函数和copy assignment 操作符有相近的代码，消除重复代码的做法是，建立一个新的成员函数给两者调用。这样的函数往往是private 而且常被命名为init。


## 令operator = 返回一个reference to *this
令赋值操作符返回一个reference to *this。这样是为了能够写成连锁形式，比如

```cpp
int x, y, z;
x = y = z = 15;

class Widget {
    public:
       Widget& operator = (const Widget& rhs) {
            return *this;
        }
        Widget& operator+= (const Widget& rhs) {

        };

};
```

## 在operator = 中处理"自我赋值"

### 要点

* 确保对象自我赋值的时候，operator = **有良好的行为**，其中的技术包比较**来源对象和目标对象的地址**，精心周到的语句顺序，copy and swap。
* 确保任何函数如果操作一个以上的对象，而其中多个对象是同一个对象时，其行为仍然正确。

自我赋值是自己赋值给自己，看起来愚蠢，但是是合法的。比如：

```cpp
a[i] = a[j]; //潜在的自我赋值
*px = *py; //潜在的自我赋值
```

这里有一种错误的写法:

```cpp
class Bitmap {
    ....
};
class Widget {
    ...
     private:
        Bitmap *pb;
};

Widget&  Widget::operator =(const Widget rhs) {
    delete pb;
    pb = new Bitmap(*rhs.pb);
    return *this;
}
```
如果*this 和 rhs是同一个对象，那么delete pb销毁的就不只是当前对象bitmap，连rhs的bitmap都给销毁了，可以这样来解决这个问题：

```cpp
Wideget& Widget::operator = (const Widget&　rhs)  {
    if(this == &rhs)
        return
    delete pb;
    pb = new Bitmap(*rhs.pb);
    return *this;
}
```
但是上面的代码不具有异常安全性，如果 new Bitmap发生了异常，Widget**最终会指向一块被删除的Bitmap**，这样的指针是有害的。那我们就写一个异常安全的代码：

```cpp
Widget& Widget operator = (const Widget& rhs) {
    Bitmap* pOrigin  = pb;
    pb = new Bitmap(*rhs.pb);
    delete pOrigin;
    return *this;
}
```
原理就是：先复制资源，然后再删除。另外一种方法是基于**copy-swap的技术**，我们来看一看：

```cpp
Widget& Widget::operator=(const Widget& rhs) {
    Widget temp(rhs); // 为rhs数据制作一份复件
    swap(temp); // 将this数据和上述复件的数据交换
    return *this;
}
```

## 以对象管理资源

为防止资源泄露，请使用**RAII对象**，它们在**构造函数中获得资源并在析构函数中释放资源**。

## 在资源管理类中小心的coping的行为

复制RAII对象**必须一并复制它所管理的资源**，所以资源的copying行为决定了RAII对象的copying行为。

## 在资源管理类中提供对原始资源的访问

* API往往要求访问原始资源，所以每一个RAII 类应该提供一个取得其所管理资源的办法。
* 对原始资源的访问可能经过显示转换或隐式转换。但是显示转换比较安全，隐式转换对客户比较方便。

## 成对使用new 和delete时采取相同的形式

* 如果你在new表达式中使用[]，那么必须在相应的delete表达式中也是用[]，如果你在new表达式中不使用[]，那么在delete表达式中不要使用[]


## 设计类犹如设计type

设计类的时候要考虑的事情如下：

* 新type的对象应该如何被创建和销毁 ？
* 对象的初始化和对象的赋值应该有怎么样的差别？
* 新type的对象如果以以传值传递意味着什么？
* 什么是新type的合法值？
* 你的新type有多么一般化？

## 宁以传递const值的引用替换传值


* 尽量以传递const值的引用来代替传值，前者通常比较高效，并且避免切割问题。
* 以上规则并不适用于内置类型，以及STL的迭代器和函数对象。它们以传值的方式更为合适

## 必须返回对象的时候，别妄想返回其引用

*  在必须返回新对象的时候，绝不要返回指针或引用指向一个局部栈对象，或返回引用指向一个堆对象，或返回引用或指针指向一个局部静态对象。


## 宁以non-member，non-friend 替换member函数

宁可拿**non-member non-friend 函数**替换**成员函数**。这样做可以增加封装性，包裹性和机能扩充性。假如你要写一个浏览器类，其中有清除历史记录，url、cookie等。那么可以这样设计。

```cpp
class WebBrowser {
  public:
        void clearCache();
        void clearHistory();
        void clearCookies();
};
```
那么你可能想提供一个函数来清除所有的这些动作：

```cpp
class WebBrowser {
public:
    void clearEveryThing():

};
```

也可以这样写：

```cpp
void clearBrowser(WebBroser& web) {
    web.clearCache();
    web.clarHistory();
    web.clearCookies();
}
```
哪种方式好呢？ 面向对象告诉我们，数据应该和操作绑定一起，这意味着member函数是好的选择，实际上这样是不对的。此外，提供non-member函数可允许对WebBrowser提供相关的包裹弹性，导致较低的编译依赖。

我们可以认为越多的东西被封装，越少的人就可以看到它，越少的人看到，我们就可以有越多的弹性去改变。这就是我们推崇封装的原因。我们建议成员变量为private，这样就会有越少的函数来访问。同样，在non-member，non-friend函数和member函数提供相同的功能的情况下，要我们去选择，我们应该选择non-member和non-friend的函数，因为它不增加`能够访问private成员函数的数量`。

这并不意味着这个non-member non-friend函数不可以是别的类的成员。我们可以令clearEveryThing成为某个工具类的static成员。在C++中，比较自然的做法是声明一个名字空间，将这些non-member函数位于WebBroser的名字空间中。

```cpp
namespace WebBrowserStuff {
    class WebBrowser {

    };
    void clearBrowser(WebBrowser& web);
}
```

当然，我们可以有很多关于浏览器的便利函数，有的是关于书签的，有的是关于cookie的。对于这些便利函数的管理，我们可以这样做：

```cpp
namespace WebBrowserStuff {
    class WebBrowser｛
    }；
}

// 头文件 "webbrowserbookemarks.h"
namespace WebBrowser {
    // 书签相关的便利函数
}
// 头文件"webbrowsercookies.h"
namespace WebBrowserStuff {
// cookie相关的头文件
}
```
这就是标准库的做法，可以降低编译依赖。

### 为什么这样写可以提供扩展性
可以考虑，我们要写一个关于**影像下载的函数**，我们只需在WebBrowserStuff的命名空间中，声明这些变量函数就可以与原来的便利函数一起使用。而class的定义对于客户是不能扩展到。


## 若所有参数都需要类型转化，请为此采用non-member函数

### 要点

## 尽量少做转型的操作

本条款论证了为什么要尽量少做类型转换，并告诉读者，如果必须要进行类型转换，有哪些注意事项：

常见的有三种类型转换方式

* C风格：(T)expression
* 函数风格：T(expression)
* C++ style cast

举例子：

* const_cast<T>(expression) : 移除变量的const属性
* dynamic_cast<T>(expression) : 安全向下转型，即：基类指针/引用到派生类指针/引用的转换。如果源和目标类型没有继承/被继承关系，编译器会报错；否则必须在代码里判断返回值是否为NULL来确认转换是否成功。
* reinterpret_cast<T>(expression)：底层转换
* static_cast<T>(expression)：强迫隐式转换，如，将non-const对象转换为const对象，将int转换为double类型。


## 考虑写出一个不抛出异常的swap函数
**swap**是STL中的标准函数，用于**交换两个对象的数值**。后来swap成为异常安全编程（exception-safe programming）的脊柱，也是实现**自我赋值**（条款11）的一个常见机制。swap的实现如下：

```cpp
namespace std {
    template<typename T>
    void swap(T& a, T& b) {
        T temp(a);
        a=b;
        b=temp;
    }
}
```

只要**T支持copying函数**（copy构造函数和copy assignment操作符）就能**允许swap函数&&。这个版本的实现非常简单，a复制到temp，b复制到a，最后temp复制到b。

但是对于某些类型而言，这些复制可能无一必要。例如，class中含有指针，**指针指向真正的数据**。这种设计常见的表现形式是所谓的“pimpl手法“。如果以这种手法设计Widget class

```cpp
class WidgetImpl{
public:
    ……
private:
    int a,b,c;              //数据很多，复制意味时间很长
    std::vector<double> b;
    ……
};
```

下面是pimpl实现

```cpp
class Widget{
public:
    Widget(const Widget& rhs);
    Widget& operator=(const Widget& rhs) {
        ……          //  复制Widget时，复制WidgetImpl对象
        *pImpl=*(ths.pImpl);
        ……
    }
    ……
private:
    WidgetImpl* pImpl;//指针，含有Widget的数据
};

```

如果**置换两个Widget对象值，只需要置换其pImpl指针**，但STL中的**swap算法不知道这一点，它不只是复制三个Widgets，还复制WidgetImpl对象，非常低效**。 我们希望告诉std::swap，当Widget被置换时，只需要**置换其内部的pImpl指针即可**，下面是基本构想，但这个形式无法编译（不能访问private）。

```cpp
namespace std{
    template<>      //  这是std::swap针对T是Widget的特换版本，
    void swap<Widget>(Widget& a, Widget& b) //目前还无法编译
    {       // 只需要置换指针
        swap(a.pImpl, b.pImpl);
    }
}
```
其中template<>表示std::swap的一个**全特化**（total template specialization），函数名之后的**<Widget>表示这一特化版本系针对T是Widget而设计的**。我们被**允许改变std命名空间的任何代码**，但是可以为标准的template编写特化版本，使它专属于我们自己的class。

上面函数试图访问private数据，因此无法编译。我们可以**将swap函数声明为friend**，但这个和以往有点不同。可以令Widget的swap函数为public，然后将std::swap特化

```cpp
calss Widget{
public:
    ……
    void swap(Widget& other)
    {
        using std::swap; // 这个声明有必要
        swap(pImpl, other.pImpl);
    }
    ……
}；
namespace std{
    template<> //修订后的swap版本
    void swap<Widget>(Widget& a, Widget& b) {
        a.swap(b);  //调用其成员函数
    }
}
```
这个做法还跟STL容器保持一致，因为STL容器**也提供public swap和特化的std::swap（用来调用前者）**。 刚刚假设Widget和WidgetImpl都是class，而不是class template，如果是template时：

```cpp
template<typename T>
class WidgetImpl{……};
template<typename T>
class Widget{……};
```

可以在Widget内或WidgetImpl内放个swap成员函数，像上面一样。但是在特化std:swap 时会遇到麻烦

```cpp
namespace std{
    template<typename T>
    void swap<Widget<T> >(Widget<T>& a,//不合法，错误
                          Widget<T>& b)
    {
        a.swap(b);
    }
}
```
开起来是对的，我们偏特化一个函数模板，但是C++只允许对**类模板进行偏特化**。但是，当我们偏特化一个函数模板，我们可以简单的**添加一个重载版本**。

```
namespace std {
    template<typename T>
    void swap(Widget<T>& a), Widget<T>& b) {
         a.swap(b);
    }
}
```
一般来说，重载函数模板是没有问题的，但是std是一个特殊的命名空间，其管理规则比较特殊：客户可以**全特化std中的模板，但不可以添加新的模板**，所以我们定义一个新的命名空间，然后其中定义**swap模板函数**。

```cpp
namespace WidgetStuff {
    template<typename T>
    class Widget {   ...    }
    ...
    template<typename T>
    void swap(Widget<T>& a, Widget<T>& b) {
       a.swap(b);
    }
}
```
* 当std::swap对你的类型效率不高的时，提供一个**swap成员函数**，并确定这个函数不抛出异常。
* 如果你提供一个member swap，也该提供一个non-member swap 用来调用前者。对于classes而不是templates而言，请**特化std::swap**。
* 调用swap时，应针对std::swap使用using 声明式，然后调用swap并且不带任何命名空间资格修饰。
* 为用户自定义类型进行std::templats全特化是最好的，但最好不要尝试在std内添加某些对std而言全新的东西。

## 尽量延后变量定义式的出现时间

### 要点

*  尽可能延后变量定义式的出现，这样可以增加程序的清晰度并改善程序的效率。

## 尽量少做转型操作

* 如果可以，尽量避免转型，特别是注重效率的代码中**避免dynamic_casts**，如果有个设计需要转型操作，那么尝试设计无需转型的设计。
* 如果转型是必要的，试着隐藏到某个函数的背后，客户随后可以调用该函数，而不需要将转型放进它们自己的代码
* 宁可使用C++style的转型，不要使用旧式转型。前者有着很容易的辨识。

## 避免返回Handles指向对象的内部成分

* **避免**返回Handles（包含引用，迭代器，指针）指向**对象内部**。这样就可以**增加封装性**，帮助成员函数的行为像一个const。


下面看看这个例子

```cpp
class Point {
    public:
        Point(int x, int y);
        void setX(int newVal);
        void setY(int newVal);
};
struct RectData {
    Point Ulhc;
    Point lrhc;
};

class Rectangle {
    ...
private:
    std::tr1::shared_ptr<RectData> pData;
};
```
为了让使用矩形的客户能够计算矩形的范围，这个函数将会提供**upperLeft函数**和**lowerRight函数**。Point是一个用户自定义类型，所以根据上面的条款的忠告，这些函数将会返回引用，代表底层的Point对象。

```cpp
class Rectangle {
    public:
        Point& upperLeft() const {
            return pData->ulhc;
        }
        Point& lowerRight() const {
            return pData->lrhc;
        }
};
```

这样的设计是可以通过编译，但是**设计却是错误的**。一方面upperLeft函数和lowerRight函数被声明为const的成员函数，目的是为了提供一个Rectangle相关的坐标点方法，而不是让用户去修改Rectangle。


## 为异常安全而努力是值得的

## 透彻了解inlining的里里外外

inline函数可免除函数调用成本，提高程序执行效率，但它也会带来负面影响：

* 增大**目标代码的大小**，有时候会非常庞大，需要动用虚存，这将大大降低程序执行速度。
* inline 函数**无法随着程序库的升级而升级**。换句话说如果f 是程序库内的一个inline 函数，客户将”f 函数本体”编进其程序中，一旦程序库设计者决定改变f ，所有用到f 的客户端程序都必须重新编译。总之，将大多数inlining 限制在**小型、被频繁调用的函数身上才是最明智的选择**（根据80-20经验准则，80%的时间花在20%的函数上）。

## 确定你的public继承塑造出来的是is-a模型

* public 继承意味着 is-a，适用于base class身上的**每件事情一定也要适用于子类身上**，因为每个子类对象都是一个**基类对象**。
* virtual 函数表示接口必须继承、non-virtual表示接口和思想必须被继承。
## 避免遮掩继承而来的名称

下面的代码因为存在遮掩的问题

```cpp
int x;
void someFunc() {
    double x;
    std::cin >> x;
}
```
读取的数据时local变量x，而不是global变量x。遇到名称x的时候，它在local作用域中查找是否有什么东西带着这个名称，如果找到，就不找到其他作用域。

* 请记住子类的名称会遮掩父类的名称。在public继承下，从没有人希望如此。


## 区分接口继承和实现继承

有如下的例子

```cpp
class Shape {
    public:
        virtual void draw() const = 0;
        virtual void error(const std::string& msg);
        int objectId() const;
};

class Rectangle: public Shape {

};
class Ellipse: public Shape {

};
```
Shape是抽象类，它的 pure virtual 函数draw 使它成为一个抽象类。意味着不能够创建一个Shape 类的实体，只能创建子类的实体。

* 成员函数的接口总是会被继承。因为public继承是is-a继承，所以对父类为真的任何事情一定对其子类的为真。
* 声明一个纯虚函数的目的是为了让子类只继承函数接口。
* 声明非纯虚函数的目的，是让子类继承该函数的接口和缺省实现。
* 非虚函数具体指定了接口继承以及强制性实现继承。

## 考虑virtual函数以外的其他选择

### 使用Non-Virtual Interface 手法实现模板方式

```cpp
class GameCharacter {
public:
    int healthValue() const {
        ...
        int retVal = doHealthValue();
        ...
        return retVal;
    }
private:
    virtual int doHealthValue() const {
        ...
    }
};
```
### 借助函数指针来实现策略模式

```cpp
class GameCharacter;
int defaultHealthCalc(const GameCharactor& gc);

class GameCharacter {
    public:
        typedef int (*HealthCalcFunc)(const GameCharacter&);
        explicit GameCharacter(HealthCalcFunc hcf = defaultHealthCalc): healthFunc(hcf) {
        }
    private:
        HealthCalcFunc healthFunc;
};
```


## **绝不重新**定义继承而来的non-virtual函数

例子：

```cpp
class B {
    public:
        void mf();
};

class D: public B {

};
D x;
B* pB = &x;  //获取一个指针指向x
pB->mf();  //  调用mf
D* pD= &x; // 获取一个指针指向x
pD->mf(); //  经由指针调用mf
```
你会觉得通过对象x调用成员函数mf，由于两者所调用的成员函数相同，凭借的对象也相同，那么行为也相同，是吗？在**如下情况下**，不会是这样：

mf是一个**非虚函数**，但是子类重写了这个mf的方法。

```cpp
class D : publicＢ{
public:
    void mf() ;  //遮掩了B::mf
};
pB->mf();  //调用 B::mf
pD->mf(); //调用 D::mf
```
当mf被调用的时候，任何一个**D对象可能表现出B或D的行为**；决定因素不在对象自身，而在于**指向该对象之指针**，这就不合理。对于public继承，适用于B对象的每一件事情，都应该适用于D对象。上面违反了这个规定，D重新定义了mf，设计出现了矛盾。

* 在任何情况下都不该重新定义一个继承而来的non-virtual 函数
* 将一个**子类对象的地址**分别赋值给基类指针或者子类指针，分别调用重写的的函数，其结果不同。

## 继承体系下指针比较的含义

在C++中，一个对象**可以有多个有效地址**，   这样地址的比较**不是关于地址相同的问题**，而是**对象同一性**。

```cpp
class Shape {  ... };
class Subject { ... };
class ObservedBlob: public Shape, public Subject { ..  };
ObservedBlob *b = new ObservedBlob;
Shape* s = ob;
Subject *subj = ob;
```
我们进行如下测试：

```cpp
if( ob == s) ; // true
if (subj == ob) ; // true
```
实际上，我们发现，几种指针以及内存布局如下：


 布局1中，s和subj分别指向了**子类对应的父类部分**。而obj则完整的指定了整个子类，s和subj具有和obj不同的地址。

不管哪种布局，上面的**if语句的比较都是true**，不能够**拿s和sub比较**，因为它们**不具有继承关系**，它们具有不同的地址，但是比较的结果相同（true），这是为什么呢？这是因为编译器调整了一定的偏移量来完成的。

```cpp
ob == sub;
```
可能被翻译成：

```cpp
ob? (ob + delta  == subj) : (subj == 0);
```
也就是说，ob是**空指针**，它们就相等，否则**ob调整为基类对象**。这里可以得到一个非常重要的经验： 一般而言，当我们处理对象的指针和引用的时候，必须**小心避免失去类型信息**。


```cpp
void* v = subj;
if (ob == v); // 不相等。
```
## 通过复合塑模出has-a或根据某物实现出

* 复合的意义和public继承完全不同
* 在应用领域：复合意味着has-a。在实现领域，复合意味着is-implemented-in-terms-of（根据某物实现出）


## 指针和引用的区别

在下列情况下，你应该使用指针

* 存在不指向任何对象的可能。
* 你需要在不同的时刻指向不同的对象。
* 重载某个操作符的时候，你应该使用引用。

## 尽量使用 C++风格的类型转换

使用C风格的指针，问题

* 过于粗鲁，能允许你在任何类型之间进行转换。
* 在程序语句中难以识别。

## 要求或者禁止对象分配在堆上。

### 要求对象分配在堆上

我们必须要找到一种方法能够通过new 来创建对象，而不能够通过其他的方式，非堆对象在它们定义的时候被会自动构造并且在生存期结束的时候，能够被自动析构，所以只有想办法将构造函数和析构函数设置为非法操作。如果将两者都设置为private，没有必要，我们只需将析构函数设置为私有既可。

```cpp
class UPNumber {
public:
    UPNumber();
    UPNumber(int val);
    void destroy() const {
        delete this;
    }
private:
    ~UPNumber();
};
UPNumber n;
UPNumber* p = new UPNumber();
delete p; // error;
p->destory(); // fine
```
还有一种方法是将所有的构造函数声明为私有，包含拷贝构造函数、默认构造函数等。因为要将每个构造函数声明为私有，所以还是将析构函数来声明为private简单。

### 禁止对象分配在堆上

**禁止对象直接通过new来实例化对象**，这种禁止很简单，因为总是通过**new来创建**，所以你可以通过某种某种方法来**禁止调用new**，**new操作符的可用性，你不能够控制**，但是你能够控制operator new 函数，你可以将这个函数声明为private。

```cpp
class UPNumber {
priate:
     void* operator new(size_t size);
     void operator delete(void* ptr);
     void operator* new[](size_t size);
     void operator delete[](void* ptr);
};

UPNumber n1; //OK
static UPNumber n2; // OK
UPNumber* p = new UPNumber; //error
```
我们看到**operator new**和 **operator delele **两者都**声明在private中**，除非有具体的理由否则，我们不应该将其中一个放在private中，如果要禁止UPNumber数组分配在堆上，我们应该讲operator new[]和operator delete[] 放在private中。

### 强制使用堆对象‘
因为栈对象，在离开作用域的时候，会自动调用类的析构函数，所以我们将**析构函数声明为private**，这样在析构的时候，将会报错。从而强制用户使用堆对象。

```cpp
class OnHeap {
private:
    ~OnHeap();
pubic:
    void destroy() {
        delete this;
    }
};
```
## 名字空间

```cpp
namespace org_semantics {
    class String { .... };
    String operator+(const string& s, const string& b);
}
```
上面的名字空间的代码**是声明**，我们可以重新打开该名字空间来定义。注意这样做，可能重新打开定义了一个名字空间。

```cpp
namespace org_semantics {
    String operator+(const String&a, const String& b);
}
```
为了避免上面提到的问题，我们可以使用全名来限定定义：

```cpp
org_semantics::String org_semantcis::operator+(const semantics::String& a, const semantics::String& b);
```
如果希望使用某个名字空间的名称，我们可以使用**using指令**，

```cpp
void aFunc() {
    using namespace std;
    vector<int> a{1, 2,3};
}
```
注意我们在这里在函数作用域内使用了using 指令，其作用域一直扩展到函数结束，其他的代码就看不到了**std**，有的程序员，喜欢将using指令放在程序的开头，这时**不好的习惯**，因为其作用范围太大，跟没有使用名字空间是一样的。

## 协变返回类型
我们知道，一般来说重写的函数需要和被重写的**函数具有相同的原型**。

```cpp
class Shape {
    public:
        virtual double area() const = 0;
};
class Circle {
    public:
        virtual float area () const ; // 错误
};
```
因为重写的接口，在Circle中的返回值改成了**float**，所以不能够**重写**，但是有一类意外：就是返回的**类型为子类的指针和引用**，这种情况是允许的。

```cpp
class Shape {
    public:
       virtual Shape* clone() const =0;
};
class Circle｛
    public:
        virtual Shape* clone() const ; // OK
};
```

## 制造抽象基类
在正常情况下，我们可以定义一个纯虚函数来使**一个类**变成**抽象类**，也可以通过继承使得类获取纯虚函数，使得类变成抽象基类。然而，有时候，我们抽象的时候，找不到**合适的纯虚函数**。那么，我们应该怎样定义一个**行为类似于抽象基类呢**？

这里，可以通过**确保类中没有公有的构造函数来模拟抽象基类的性质**。这就意味着我们**必须显示的声明一个构造函数**，否则编译器会为我们生成默认构造函数。

```cpp
class ABC {
    public:
        virtual ~ABC();
    protected:
        ABC();
        ABC(const ABC&);
};
class D: public ABC {
};
```
这两个构造函数被声明为受保护的，是为了**让派生类的构造函数能够使用**，同时阻止**创建独立的ABC对象**。

```cpp
void func1(ABC);
void func2(ABC&);
ABC a; // 错误
D d;
func1(d); // 错误，复制构造函数式受保护的。
func2(d); // 正确
```
另一种使一个类成为抽象类，可以声明一个**virtual虚构函数**为纯虚的，析构函数是最佳的候选者。

```cpp
class ABC {
pubic:
    virtual ~ABC() = 0;
};
```

## placement new

placement new 是**operator new**的一个**重载版本**，和operator new不同的是，**语言禁止用户替换placement new**，而普通的**opeator new **和**opeartor delete**是可以被替换掉的。placement new 直接**忽视第一个表示大小的实参**，直接返回第二个参数，它允许我们在**特定的位置放置对象**。

```cpp
// placement new
void* operator new(size_t size, void *p ) throw {
    return p;
}
```
placement new 的使用

```cpp
class SPort {..... };
const int comLoc = 0x004000;
void* comAddr = reinterpret_cast<void *>(comLoc);
SPort* com1 = new (comAddr) Sport; // placement new 的使用。
```
从上，我们知道placement new是不会分配任何存储，仅仅是返回一个指向**已经分配好空间的指针**。所以，不能够对其调用delete操作：

```cpp
delete com1;  // 错误
```
如果要销毁placement 实例化的对象，我们应该调用其析构函数：

```cpp
com1->~SPort();
```
然而，自己主动的调用析构函数**往往容易出错**，常常导致一个对象析构多次，或者根本没有**析构**，只有在有必要的时候，才这样设计。


## 特定于类的内存管理

如果你不希望你的类使用全局的**operator new**和**operator delete**，那么你可以在自己的类中，显示的定义属于自己的operator new 和 delete。

```cpp
class Hanle {
    public:
        void* operator new(size_t );
        void operator delete(void *);
};
```
在**new 操作符进行对象的分配**的时候，**首先查看Handle的作用域内是否定义了operator new**，如果没有找到，那么就使用全局作用域**operator new**。你定义了operator new ，那么一般要定义operator delete。成员operator new 均为**静态成员函数**。因为如果不是静态成员函数，因为operator new需要调用在对象构造之前，operator delete 需要调用在对象析构之后，若是一般的成员函数，对象在构造之前，是**不有效的**。静态成员函数中**没有this指针**。operator new 和 operator delete 是可以被子类继承的，当然如果**子类重写了它们，将调用子类的**。

```cpp
class MyHandle : public Handle {

};
Hanle* my = new MyHandle();  // use Handle::operator new
delete my; // use Handle::operator delete
```
当然，作为基类的**Handle**需要有一个虚析构函数，否则**不会调用子类的析构函数**和**operator delete**。一个常见的误解是**使用new操作符就意味着在堆中分配内存，实际上不是如此**，new操作符只是意味着**operator new 将被调用**，而且返回一个**指向某块内存的地址**。全局的**operator new**意味着在堆中申请内存，但是类中的operator new对分配的内存**并没有任何限制**。可以使**静态分配的内存**，也可以是**容器内部**。

## 重载operator++的前缀和后缀模式

```cpp
class Number {
public:
    Number& operator++ () // prefix ++ {
        // Do work on this. (increment your object here)
        return *this;
    }
    // You want to make the ++ operator work like the standard operators
    // The simple way to do this is to implement postfix in terms of prefix. //
    Number operator++ (int) // postfix ++ {
        Number result(*this); // make a copy for result
        ++(*this); // Now use the prefix version to do the work
        return result; // return the copy (the old) value.
    }
};
```

## 使用带检查的STL实现
带检查的STL实现有哪些？

## 用算法代替手工编写的循环
调用算法时，应该考虑编写自定义函数对象来封装所需要的逻辑。要抛弃那种处理每个元素的循环式思路。算法的效率也可能比循环好，手写迭代器可能出现迭代器无效的情况。

几个例子：

```cpp
deque<double>::iterator current = d.begin();
for (size_t i = 0; i < max; ++i) {
    current = d.insert(current, data[i] + 41);
    ++current;
}
```
使用算法：

```cpp
transform(data, data + max, insterter(d, d.begin()),  _1 + 41);
```
我们看看inserter的定义：


和 std::instert_iterator的定义


## 使用正确的查找算法
对于查找算法有：
* find/ find_if
* count/count_if
* binary_search
* lower_bound
* upper_bound
* equal_range
