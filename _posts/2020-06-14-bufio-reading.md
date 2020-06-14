---
title: bufio的源码阅读
layout: post
date: 2020-06-11
categories: golang
---

## 介绍
这个package提供带有缓冲的io。

## Reader

```go
// Reader implements buffering for an io.Reader object.
type Reader struct {
    buf          []byte
    rd           io.Reader // reader provided by the client
    r, w         int       // buf read and write positions
    err          error
    lastByte     int // last byte read for UnreadByte; -1 means invalid
    lastRuneSize int // size of last rune read for UnreadRune; -1 means invalid
}
```

![bufio-structure](https://user-images.githubusercontent.com/4377940/84590475-9acac300-ae69-11ea-9ab5-43178b8cb741.png)

这个struct包含了最后一次读取的字节，和分配的buf[]byte，其中r和w为两个索引，分别用来表示已经读取的位置，和下次需要从rd中写的位置。

![bufio](https://user-images.githubusercontent.com/4377940/84590184-e6c83880-ae66-11ea-8cdf-c425a44ae789.png)


```go
// 从IO设备，文件描述符中read到p中
func (b *Reader) Read(p []byte) (n int, err error) {
    // 查看slice的大小
    n = len(p)
    if n == 0 {
	// 如果还有剩余的，那么不返回错误
	if b.Buffered() > 0 {
		return 0, nil
	}
	// 返回上次的读取的错误
	return 0, b.readErr()
    }

    // 已从buffer中读完了
    if b.r == b.w {
	// 上一次读取失败，将结果返回
	if b.err != nil {
		return 0, b.readErr()
	}
	// 如果要读的size比buffer还要大，直接从原有的reader中读
	if len(p) >= len(b.buf) {
		// Large read, empty buffer.
		// Read directly into p to avoid copy.
		// 再一次从数据源中读
		n, b.err = b.rd.Read(p)
		if n < 0 {
			panic(errNegativeRead)
		}
		// 读出来了
		if n > 0 {
			b.lastByte = int(p[n-1])
			b.lastRuneSize = -1
		}
		return n, b.readErr()
	}
	// len(p) < len(b.buf)
	// 将buf重置
	// 再一次从数据源中读
	// One read.
	// Do not use b.fill, which will loop.
	b.r = 0
	b.w = 0
	n, b.err = b.rd.Read(b.buf)
	if n < 0 {
		panic(errNegativeRead)
	}
	if n == 0 {
		return 0, b.readErr()
	}
	b.w += n
    }

    // copy as much as we can
    // 复制到到用户的空间
    n = copy(p, b.buf[b.r:b.w])
    b.r += n
    // 更新上一次读取的位置
    b.lastByte = int(b.buf[b.r-1])
    b.lastRuneSize = -1
    return n, nil
}
```
## writer
带buffer的writer是将p []byte中的数据，写入内部的bufer中，等到内部的buffer满后，才执行一次真正的write写到设备或者是文件描述符上。
```go
type Writer struct {
    err error
    buf []byte
    n   int
    wr  io.Writer
}
func NewWriterSize(w io.Writer, size int) *Writer {
    // Is it already a Writer?
    b, ok := w.(*Writer)
    // 直接返回自己
    if ok && len(b.buf) >= size {
	    return b
    }

    // 选择默认的size
    if size <= 0 {
	    // 4KB
	    size = defaultBufSize
    }
    return &Writer{
	    buf: make([]byte, size),
	    wr:  w,
    }
}

// 将p中的内容写到buffer中
func (b *Writer) Write(p []byte) (nn int, err error) {
    // p的size大于可用的空间，那么
    for len(p) > b.Available() && b.err == nil {
	var n int
	// 如果之前的bufer并没有写入任何数据且
        // 如果p的剩余字节数大于buffer，那么直接写到设备中。
	if b.Buffered() == 0 {
	    // Large write, empty buffer.
	    // Write directly from p to avoid copy.
	    // 那么使用buffer，直接写
	    n, b.err = b.wr.Write(p)
	} else {
	    // 将目前的buffer填满
	    n = copy(b.buf[b.n:], p)
	    b.n += n
	    // 执行一次真正的写操作，让buffer空出来
	    b.Flush()
	}
	nn += n
	p = p[n:]
    }
    if b.err != nil {
	return nn, b.err
    }
    // 将目前的可以容纳的，p中的数据copy到buf中
    n := copy(b.buf[b.n:], p)
    b.n += n
    nn += n
    return nn, nil
}

```
