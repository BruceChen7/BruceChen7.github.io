---
title: failure的源码学习
layout: post
date: 2020-06-11 16:03:54
---
## 源码
来自[这里](https://github.com/morikuni/failure)，其给error提供了一种错误码的方式来表示错误，并且与golang中的error能够兼容。具体的API可以见下面的测试用例

## 测试用例

```golang
// 创建错误码
// 内置StringCode
var NotFound failure.StringCode = "NotFound"

// 根据错误码来创建error
err := failure.New(NotFound)

// 判断是否为相关错误码
if failure.Is(err, NotFound) { // true
        r.WriteHeader(http.StatusNotFound)
}

const (
    A failure.StringCode = "A"
    B failure.StringCode = "B"
)

// 创建一个错误
errA := failure.New(A)
// translate的含义是将errA错误，使用B的错误码，生成一个新的错误
errB := failure.Translate(errA, B)
// wrapper C错误
errC := failure.Wrap(errB)


// 都是A错误码
shouldEqual(t, failure.Is(errA, A), true)
// 是B错误码
shouldEqual(t, failure.Is(errB, B), true)
// 是B错误码
shouldEqual(t, failure.Is(errC, B), true)

// errA有两个错误码A, B错误码中的一个就可以
shouldEqual(t, failure.Is(errA, A, B), true)
// errB有两个错误码A, B错误码中的一个就可以
shouldEqual(t, failure.Is(errB, A, B), true)
// errC有两个错误码A, B错误码中的一个就可以
shouldEqual(t, failure.Is(errC, A, B), true)

// 查看底层的错误是否是相关的错误码
shouldEqual(t, failure.Is(errA, B), false)
shouldEqual(t, failure.Is(errB, A), false)
shouldEqual(t, failure.Is(errC, A), false)
```

## 重要的接口和实现

```golang
// 从错误码中创建error类型的值
func New(code Code, wrappers ...Wrapper) error {
    // Custom是一个函数，其不需要对外暴露
    return Custom(Custom(&withCode{code: code}, wrappers...), WithFormatter(), WithCallStackSkip(1))
}

// 接口，用来包裹错误，返回另一个错误
type Wrapper interface {
    // WrapError should wrap given err to append some
    // capability to the error.
    WrapError(err error) error
}


type withCode struct {
    code Code
    // 底层的error
    underlying error
}

// Wrapper是一个interface, 也产生一个Wrapper，这样可以在Cunstom的构造函数中使用
func WithCode(code Code) Wrapper {
     // 强制转换为一个函数指针
     return WrapperFunc(func(err error) error {
          return &withCode{code, err}
     })
}


// 函数指针
// WrapperFunc is an adaptor to use function as the Wrapper interface.
type WrapperFunc func(err error) error

// 函数指针实现WrapperError接口
// WrapError implements the Wrapper interface.
// 类似闭包的手法
func (f WrapperFunc) WrapError(err error) error {
        return f(err)
}


// 用来包裹error
// wrappers是一系列实现WrapError的接口
func Custom(err error, wrappers ...Wrapper) error {
    if err == nil {
        return nil
    }
    // To process from left to right, iterate from the last one.
    // Custom(errors.New("foo"), Message("aaa"), Message("bbb")) should be "aaa: bbb: foo".
    // 层层包裹错误
    for i := len(wrappers) - 1; i >= 0; i-- {
        err = wrappers[i].WrapError(err)
    }
    return err
}
```

**创建带有code的error---failure.New**
具体的流程如下：

<p align="center"><img src="/assets/img/failure-learning/failure.jpg" width="100%" height="100%"></p>


**error是否带有某个code---failure.Is**

```golang
func Is(err error, codes ...Code) bool {
    if len(codes) == 0 {
        return false
    }
    // extract err code
    c, ok := CodeOf(err)
    if !ok {
        // continue process (don't return) to accept the case Is(err, nil).
        c = nil
    }

    for i := range codes {
        // 比较code之间是否相等
        if c == codes[i] {
            return true
        }
    }
    return false
}

// CodeOf extracts an error code from the err.
// 从错误中获取错误码
func CodeOf(err error) (Code, bool) {
    if err == nil {
        return nil, false
    }
    // 创建一个迭代器，实际上也就是一个error
    // 向下迭代error
    i := NewIterator(err)
    for i.Next() {
        // 实现了NoCode
        if noCode, ok := i.Error().(interface{ NoCode() bool }); ok && noCode.NoCode() {
            return nil, false
        }

       var c Code
       // 普通的error也实现了As，那么返回对应的凑到
       if i.As(&c) {
           return c, true
       }
    }
    return nil, false
}                                                                                                                                   ```
```
在自定义几种的error中，都有定义As

```golang
func (w *withCallStack) As(x interface{}) bool {...}
func (w *withContext) As(x interface{}) bool {...}
func (w *withMessage) As(x interface{}) bool {...}
```
