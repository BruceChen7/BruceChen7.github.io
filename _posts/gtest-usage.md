---
title: "Gtest的使用 "
layou: post
date: 2014-12-15 19:29:55
tags: [Tools,Unit Testing]
---

通过提出问题来开始今天的内容。

## 问题

* 怎样安装gtest?
* 如何使用gtest来进行单元测试?


## 怎样安装gtest?

* 下载包

	wget http://googletest.googlecode.com/files/gtest-1.7.0.zip


* 解压包并且编译成静态库
>	unzip gtest-1.7.0.zip
	cd gtest-1.7.0
	./configure
	make


* 安装你的头文件和库到你的系统
>	sudo cp -a include/gtest /usr/include
	sudo cp -a lib/.libs/* /usr/lib/

* 更新链接器的缓存.
>	sudo ldconfig -v | grep gtest


如果一切正常,输出如下:
>	libgtest.so.0 -> libgtest.so.0.0.0
	libgtest_main.so.0 -> libgtest_main.so.0.0.0

## 简单的测试一下

* 主函数(main.cpp)


```cpp
#include <gtest/gtest.h>
#include "test.cpp"
TEST(power,testPower)
{
    ASSERT_EQ(30,power(6));

}
int main(int argc,char **argv)
{
    testing::InitGoogleTest(&argc,argv);
    return RUN_ALL_TESTS();
}

```

* 测试函数(test.cpp)

```cpp
double power(int x)
{
    return x*x;
}
```

* 编译并且运行


```cpp
g++  main.cpp  -o main -lgtest -pthread
```


* 输出结果

```cpp
	[==========] Running 1 test from 1 test case.
	[----------] Global test environment set-up.
	[----------] 1 test from power
	[ RUN      ] power.testPower
	[       OK ] power.testPower (0 ms)
	[----------] 1 test from power (0 ms total)
	[----------] Global test environment tear-down
	[==========] 1 test from 1 test case ran. (0 ms total)
	[  PASSED  ] 1 test.
```


## 如何使用?

### 简单的概念
使用`Google Test`,你首先要写的就是一些断言,用来检查条件是否为真,断言的结果可能是`成功`,`非致命性错误`,`致命性错误`,如果是`致命性的错误`,就应该结束程序.一个`test case`包含一个或者多个测试.你应该将你的测试组成`test cases`,来反映你的测试代码的结构.当然一个测试程序能够写多个的`test cases`

### 使用断言
gtest使用断言是一些宏来模拟函数,你能够使用这些宏来测试函数或者是类.当断言失败的时候,`gtest`会产生失败的信息.

注意到不同的断言对当前测试的函数有不同的效果

* ASSERT_*,当断言失败的时候,为终止当前的测试函数
* EXPECT_*,不会产生致命的错误,因此不会终止当前测试的函数,因此它可能产生不止一个错误.

注意到`ASSERT_*`断言失败的时候是立即返回,可能不会执行后面的`clean-up code`,所以可能造成内存泄露.

### 自定义失败信息


```cpp
ASSERT_EQ(x.size(), y.size()) << "Vectors x and y are of unequal length";

for (int i = 0; i < x.size(); ++i) {
  EXPECT_EQ(x[i], y[i]) << "Vectors x and y differ at index " << i;
}
```


### 基本断言

* ASSERT_TRUE(condition);	EXPECT_TRUE(condition)	;	condition is true
* ASSERT_FALSE(condition);	EXPECT_FALSE(condition);condition is false

### 比较

* ASSERT_EQ(expected, actual);	EXPECT_EQ(expected, actual);	expected == actual
* ASSERT_NE(val1, val2);	EXPECT_NE(val1, val2);	val1 != val2
* ASSERT_LT(val1, val2);	EXPECT_LT(val1, val2);	val1 < val2
* ASSERT_LE(val1, val2);	EXPECT_LE(val1, val2);	val1 <= val2
* ASSERT_GT(val1, val2);	EXPECT_GT(val1, val2);	val1 > val2
* ASSERT_GE(val1, val2);	EXPECT_GE(val1, val2);	val1 >= val2

* 这些断言支持用户自定义类型,只要你定义了相应的`==`或者是`<`等,在这里你应该更倾向于使用`ASSERT_*`.
* ASSERT_EQ()如果用在`C string`,它比较的是它们是否有相同的地址,而不会比较其内容.所以你想比较`C string`,也就是`const char *`,就不能够使用这个来判断字符串是否相等.但是如果比较`string`类型,可以使用这个断言.上面的断言对于`string`和`wstring`同样使用.


### string比较


注意这里比较的是`c strings`(const char *),对于`string`,要使用上面的比较.

* ASSERT_STREQ(expected_str, actual_str);	EXPECT_STREQ(expected_str, actual_str); 	the two C strings have the same content
* ASSERT_STRNE(str1, str2);			EXPECT_STRNE(str1, str2);			the two C strings have different content
* ASSERT_STRCASEEQ(expected_str, actual_str);	EXPECT_STRCASEEQ(expected_str, actual_str);	the two C strings have the same content, ignoring case
* ASSERT_STRCASENE(str1, str2);			EXPECT_STRCASENE(str1, str2);			the two C strings have different content, ignoring case


* NULL 指针和 空字符串不同

### 简单的测试


```cpp
TEST(test_case_name,test_name)
    ... test body ..
```

* TEST()的宏来定义一个`test`函数,是一个不返回值的普通C++函数.
* 这里面可以使用任何C++语句
* 整个test的结果取决于断言,如果失败,则整个test失败.
* TEST()参数的`test case` 的名字，第二个参数是在`test case`中的`test name`


```cpp
// Tests factorial of 0.
TEST(FactorialTest, HandlesZeroInput)
{
  EXPECT_EQ(1, Factorial(0));
}

// Tests factorial of positive numbers.
TEST(FactorialTest, HandlesPositiveInput) {
  EXPECT_EQ(1, Factorial(1));
  EXPECT_EQ(2, Factorial(2));
  EXPECT_EQ(6, Factorial(3));
  EXPECT_EQ(40320, Factorial(8));
}

```
