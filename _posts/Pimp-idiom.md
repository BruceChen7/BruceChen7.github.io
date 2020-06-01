title: Pimp Idiom
date: 2014-12-21 17:23:23
tags: [C++]
---

下面是本文要讲的几个问题:

* 什么是`Pimpl idiom`?有什么用？
* 怎么在C++11中去实现一个?
* 一个类的哪些部分可以放到`impl`对象中去？


## 引言

在C++中，当头文件有任何的改动时，即使是头文件中的私有部分改动，所有依赖该头文件都需要重新编译．对于小项目而言，编译的时间是很短的，可以忽略不计，但是呢，对于大项目，一编译就是几个小时，这样是很浪费时间的．出现这个问题，在于C++ 没有将`接口从实现中分离`.所以在`Effective C++`中，`Scott Meyers`在`Item 31`中，这样写到,`Minimize complation depencies between files`,也就是说我们要尽量的减少文件的依赖性.问题至于C++ 没有将`接口从实现中分离`.能在头文件中减少使用头文件的，就尽量减少使用头文件．`Scott Meyers`给出了自己的建议:

* 如果使用`Object reference ` 和`object pointer`可以完成任何，就不要使用`object`.
* 如果能够，尽量使用class的声明式替换class的定义式．

所谓`class的声明式`，是指你可以这样是在头文件中区声明一个类.`class class_name`.不清楚的可以出门左转，看一下 [forward declaration](http://brucechen.gitcafe.com/C++/syntax/forward_declaration.html).

一个例子:(来自`effecive c++`)

```cpp
class Person {

    public:
        Person(const std::string& name,const Date& birthday,const Address& the_address);
        std::string name() const;
        std::string birthDate() const
        std::string address() const;

    private;
        Date the_birthday_;　//实现细目
        std::string name_;  //实现细目
        Address the_address_; //实现细目


};
```



如果就这样简单的编译这个类，是编译不通的，因为编译器不知道`Date`，`Address`类，所以你要在上面提供头文件，这就是`文件依赖`.

```cpp
#include "Date.h"
#include "Address.h"
```

上面的例子解释了什么是`文件依赖`,人们可以通过`前置声明`来部分解决文件依赖问题，但是要达到实现`接口和实现分离`，这还不够,人们提出了`Pimpl Idiom`

## 什么是Pimpl Idiom

一个表达思想的写法:

```cpp
// Pimpl idiom - basic idea
class widget {
    // :::
private:
    struct impl;        // things to be hidden go here
    impl* pimpl_;       // opaque pointer to forward-declared class
};
```


这里将所有的实现细节都交给了`struct impl`,这样对于`class Widget`接口的实现与改动都在`struct impl`中进行，对于使用这个`Widget`类头文件的用户，由于任何改动都是基于`struct impl`的，所以使用这个头文件的客户不需要重新编译他们自己的代码．只需要重新编译实现`Struct impl`的文件．


## 怎么在C++11中去实现一个

在这里，我们开发一个简单的`Employee`类，来看一下怎样实现?

```cpp
//pImp.hpp
#pragma once
#include <string>
#include <memory>
class Employee {
    public:
        Employee(std::string name, std::string id);
        ~Employee();
        Employee& operator=(Employee rhs);
        Employee(const Employee &other);
        std::string getName() const;
        std::string getId() const;
        void setName(std::string name);

    private:
        class Impl;
        std::unique_ptr<Impl> pImpl;
};


//pImp.cc
#include <string>
#include "pImp.hpp"


class Employee::Impl{
public:
    Impl(std::string name, std::string id) :name_(name), id_(id){}
    std::string getName() const;
    std::string getId()const;
    void setName(std::string name);
private:
    std::string name_;
    std::string id_;
};

std::string Employee::Impl::getId() const
{
    return id_;
}
std::string Employee::Impl::getName() const
{
    return name_;
}

void Employee::Impl::setName(std::string name)
{
    name_ = name;
}

//default destructor
Employee::~Employee() = default;

//copy constructor
Employee::Employee(const Employee & other)
{
    pImpl.reset(new Impl(*other.pImpl));
}

// assginment operator
Employee& Employee::operator=( Employee rhs)
{
    swap(pImpl, rhs.pImpl);
    return *this;
}


Employee::Employee(std::string name, std::string id) :pImpl(new Impl(name, id)){}

std::string Employee::getId() const
{
    return pImpl->getId();
}

std::string Employee::getName() const
{
    return pImpl->getName();
}

void Employee::setName(std::string name)
{
    pImpl->setName(name);
}
```

## 一个类的哪些部分可以放到`impl`对象中去

**把所有的`私有数据`(没有私有的方法)放到`impl`对象中去?**

好处是所有的前置声明所有在`private`中用到的数据．所以不需要引入额外的头文件．缺点是比如在前面的`Widget`例子中，widget接口的实现都依赖于`pimpl_->`.更重要的是，党我们给`Widget`加入或者是删除`private method`的时候，使用`Widget`类的客户还是要重新编译．

**将所有的`private member`都放到`impl`对象中去?**

解决了上面的问题，但是还是有几点要注意:

* 你不能够将虚函数放到`impl`对象中去，即使是`private`的虚函数．所以虚函数就只能放到上面的`Widget`.原因是`widget`类继承并`override`父类的`virtual function`，这个`virtual function`必须在`Widget`中，如果没有`override`，为了后面的子类能够`override`,也必须放到`Widget`
* 在`impl`对象中的函数可能要用到`Widget`中的`public function`,所以要传入`Widget 指针`给`impl`对象．
* 一种折中的方案是使用第二点，并且将放入`impl`对象中`private function`用到的`non-private`函数也放入到`impl`

* 将所有的`private non-virtual`放入到`impl`

这个就比较完美了．解决了前面的问题．

* 将所有的东西都放入`impl`，只在`Widget`类中写`public`接口．


## 怎样选择这些方法

对于一个类，我们可以化成三部分：

* 能够被客户看到的部分，也就是`public functions`．
* 能够被`子类`用到的接口，也就是`protected or virtual function member`
* 其它的部分，也就是`private and non-virtual function members`,这就是能够放入`impl`对象中的．

当且仅当第三部分，我们可以在`impl`对象中来隐藏


## 参考资料

* << effective c++>> 第三版
* [Compilation Firewalls](http://herbsutter.com/gotw/_100/)
