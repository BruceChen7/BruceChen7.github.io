title: libevent源码分析之buffer
date: 2015-03-18 16:45:01
tags: [Libevent,C,Source Code Reading]
---


## struct evbuffer_chain

* struct evbuffer_chain *next;　指向下一个这样的缓冲区
* size_t buffer_len;　//buffer的长度
* ev_misalign_t misalign; //用来对齐的字节数
* size_t off;
* unsigned flags
* int refcnt; //a number of reference to this chain
* unsigned char *buffer;



[![event buffer](/Images/SourceCode/evbuffer.jpeg)](/Images/SourceCode/evbuffer.jpeg)

<!--more-->


注意到对于缓冲区管理，需要额外的信息．libevent 对于缓冲区的管理很复杂．从上图可以看出．libevent使用链表来管理缓冲区．每个对于链表，有链表头(evbuffer)来管理．

## evbuffer_chain_new(size_t size)
创建一个`size`大小的buffer块．

```cpp
static struct evbuffer_chain *evbuffer_chain_new(size_t size)
{
    struct evbuffer_chain *chain;
    size_t to_alloc;

    //EVBUFFER_CHAIN_SIZE is the sizeof(struct evbuffer_chain);
    //
    if (size > EVBUFFER_CHAIN_MAX - EVBUFFER_CHAIN_SIZE)
        return (NULL);

    //buf 紧靠这evbuf_chain这个结构体
    //每申请一个buf,就要提供一个evbuf_chain结构体
    //+----------------------------------------------
    //+
    //+
    //+	 ----------------------------------------
    //+	 |					|
    //+	 |	evbuffer_chain 	| buffer        |
    //+	 |					|
    //+	 ----------------------------------------
    //+
    //+
    //+-----------------------------------------------
    //每一个buffer的分配都伴随着evbuffer_chain的分配
    //实际的分配的大小要加上EVBUFFER_CHAIN_SIZE
    size += EVBUFFER_CHAIN_SIZE;

    /* get the next largest memory that can hold the buffer */
    if (size < EVBUFFER_CHAIN_MAX / 2)
    {
        to_alloc = MIN_BUFFER_SIZE;
        while (to_alloc < size)
        {
            to_alloc <<= 1;
        }
    }
    else
    {
        to_alloc = size;
    }

    /* we get everything in one chunk */
    if ((chain = mm_malloc(to_alloc)) == NULL)
        return (NULL);

    //设置misalign = 0;
    // off = 0;
    memset(chain, 0, EVBUFFER_CHAIN_SIZE);

    //buffer的长度等于分配大小的长度-EVBUFFER_CHAIN_SIZE;
    chain->buffer_len = to_alloc - EVBUFFER_CHAIN_SIZE;

    /* this way we can manipulate the buffer to different addresses,
     * which is required for mmap for example.
     */
    //获取当前buffer的地址
    chain->buffer = EVBUFFER_CHAIN_EXTRA(u_char, chain);

    //当前buffer的引用为１
    chain->refcnt = 1;
    return (chain);
}
```

*  EVBUFFER_CHAIN_EXTRA(u_char,chain),返回的是sizeof(*chain)+1的地址,这个地址是chain->buffer地址。
*  mm_malloc(to_alloc): 创建一个to_alloc 大小的内存块
*  注意到每次分配的大小至少有这么MIN_BUFFER_SIZE

* 是一个宏定义，实际上是event_mm_malloc(sz),所做的事情是当用户自己定义了内存分配函数，通过函数指针则调用，如果没有则malloc.

## evbuffer_chain_free(struct evbuffer_chain *chain)


这是对内存块节点的删除．由于是多线程下的管理，所以要有同步和互斥的操作．当然，这是在内部的函数，不能够被用户使用. 调用者要进行同步和互斥操作．关键代码:

```cpp
static inline void evbuffer_chain_free(struct evbuffer_chain *chain)
{
    EVUTIL_ASSERT(chain->refcnt > 0);
    //有被多余一个的chain引用
    //不能够释放
    if (--chain->refcnt > 0) {
        /* chain is still referenced by other chains */
        return;
    }

    if (CHAIN_PINNED(chain)) {
        /* will get freed once no longer dangling */
        chain->refcnt++;
        chain->flags |= EVBUFFER_DANGLING;
        return;
    }

    /* safe to release chain, it's either a referencing
     * chain or all references to it have been freed */
    if (chain->flags & EVBUFFER_REFERENCE) {
        struct evbuffer_chain_reference *info =
            EVBUFFER_CHAIN_EXTRA(
                struct evbuffer_chain_reference,
                chain);
        if (info->cleanupfn)
            (*info->cleanupfn)(chain->buffer,
                chain->buffer_len,
                info->extra);
    }
    if (chain->flags & EVBUFFER_FILESEGMENT) {
        struct evbuffer_chain_file_segment *info =
            EVBUFFER_CHAIN_EXTRA(
                struct evbuffer_chain_file_segment,
                chain);
        if (info->segment) {

            evbuffer_file_segment_free(info->segment);
        }
    }
    if (chain->flags & EVBUFFER_MULTICAST) {
        struct evbuffer_multicast_parent *info =
            EVBUFFER_CHAIN_EXTRA(
                struct evbuffer_multicast_parent,
                chain);
        /* referencing chain is being freed, decrease
         * refcounts of source chain and associated
         * evbuffer (which get freed once both reach
         * zero) */
        EVUTIL_ASSERT(info->source != NULL);
        EVUTIL_ASSERT(info->parent != NULL);
        EVBUFFER_LOCK(info->source);
        evbuffer_chain_free(info->parent);
        evbuffer_decref_and_unlock_(info->source);
    }

    mm_free(chain);
}
```

几点要学习的地方:

* 如何表达一个缓冲区是只读和制只写的语义? 这里libevent是设置了一个标志位(在evbuffer_chain中的flag位).那么如何表达只读和只写呢？

下面看看libevent代码


```
/**< A chain used for a file segment */

#define EVBUFFER_FILESEGMENT	0x0001
/**< a chain used with sendfile */
#define EVBUFFER_SENDFILE	0x0002
/**< a chain with a mem reference */
#define EVBUFFER_REFERENCE	0x0004
/**< read-only chain */
#define EVBUFFER_IMMUTABLE	0x0008
/** a chain that mustn't be reallocated or freed,
 *or have its contents
 * memmoved, until the chain is un-pinned. */
#define EVBUFFER_MEM_PINNED_R	0x0010
#define EVBUFFER_MEM_PINNED_W	0x0020
#define EVBUFFER_MEM_PINNED_ANY \
(EVBUFFER_MEM_PINNED_R|EVBUFFER_MEM_PINNED_W)
/** a chain that should be freed, but can't be freed until it is
* un-pinned. */
#define EVBUFFER_DANGLING	0x0040
/** a chain that is a referenced copy of another chain */
#define EVBUFFER_MULTICAST	0x0080
```

所以这些缓冲区的性质都是通过一个十六进制数相应的符号位来设置。比如说：`EVBUFFER_FILESEGMENT`这个是使用最低位为1表示。如何查看相应的位是否设置呢？

```c
#define CHAIN_PINNED(ch) (((ch)->flags & EVBUFFER_MEM_PINNED_ANY) != 0 )
```


所以只需要与相应的位相与，看是否为0即可。


## 创建一个缓冲区
```cpp
struct evbuffer * evbuffer_new(void)
{
    struct evbuffer *buffer;

    buffer = mm_calloc(1, sizeof(struct evbuffer));
    if (buffer == NULL)
        return (NULL);

    LIST_INIT(&buffer->callbacks);
    buffer->refcnt = 1;
    buffer->last_with_datap = &buffer->first;

    return (buffer);
}
```
## 链表尾添加数据

我们应该注意到这个函数将数据复制到内存的缓冲区的尾部，或者复制到新建一个`evbuffer_chain`

```cpp
int evbuffer_add(struct evbuffer *buf, const void *data_in, size_t datlen)
{
    struct evbuffer_chain *chain, *tmp;
    const unsigned char *data = data_in;
    size_t remain, to_alloc;
    int result = -1;
    // 加锁，线程安全
    EVBUFFER_LOCK(buf);
    // 冻结缓冲区尾部，禁止追加数据
    if (buf->freeze_end) {
        goto done;
    }
    /* Prevent buf->total_len overflow */
    if (datlen > EV_SIZE_MAX - buf->total_len) {
        goto done;
    }

    //最后的内存块节点
    chain = buf->last;

     //第一次插入数据的时候
    if (chain == NULL) {
        chain = evbuffer_chain_new(datlen);
        if (!chain)
            goto done;
        evbuffer_chain_insert(buf, chain);
    }
        //如果最后一个节点内存是可写的
    if ((chain->flags & EVBUFFER_IMMUTABLE) == 0)
    {
        /* Always true for mutable buffers */
        EVUTIL_ASSERT(chain->misalign >= 0 &&
            (ev_uint64_t)chain->misalign <= EVBUFFER_CHAIN_MAX);
        //节点内存中剩余的内存空间
        remain = chain->buffer_len - (size_t)chain->misalign - chain->off;

        //如果容得下要使用的内存
        //复制内容，并且更新节点的信息
        if (remain >= datlen) {
            /* there's enough space to hold all the data in the
             * current last chain */
            memcpy(chain->buffer + chain->misalign + chain->off,
                data, datlen);
            chain->off += datlen;
            buf->total_len += datlen;
            buf->n_add_for_cb += datlen;
            goto out;
        }
        //如果当前的节点的buffer能够修改
        //通过调整内存也能够放的下，那么也复制数据内容
        else if (!CHAIN_PINNED(chain) && evbuffer_chain_should_realign(chain, datlen))
        {
            /* we can fit the data into the misalignment */
            evbuffer_chain_align(chain);

            memcpy(chain->buffer + chain->off, data, datlen);
            chain->off += datlen;
            buf->total_len += datlen;
            buf->n_add_for_cb += datlen;
            goto out;
        }
    }
    else
    {
        /* we cannot write any data to the last chain */
        remain = 0;
    }

    //上面条件都不满足的时候，我们执行下面的代码
    //创建一个新的evbufer_chain

    /* we need to add another chain */
    to_alloc = chain->buffer_len;

    //当最后evbuffer_chain的缓冲区小于等于2048时，那么新建的evbuffer_chain的
        //大小将是最后一个节点缓冲区的2倍。
    if (to_alloc <= EVBUFFER_CHAIN_MAX_AUTO_SIZE/2)
        to_alloc <<= 1;
    if (datlen > to_alloc)
        to_alloc = datlen;
    tmp = evbuffer_chain_new(to_alloc);
    if (tmp == NULL)
        goto done;

    if (remain) {
        memcpy(chain->buffer + chain->misalign + chain->off,
            data, remain);
        chain->off += remain;
        buf->total_len += remain;
        buf->n_add_for_cb += remain;
    }

    data += remain;
    datlen -= remain;

    memcpy(tmp->buffer, data, datlen);
    tmp->off = datlen;
    evbuffer_chain_insert(buf, tmp);
    buf->n_add_for_cb += datlen;

out:
    evbuffer_invoke_callbacks_(buf);
    result = 0;
done:
    EVBUFFER_UNLOCK(buf);
    return result;
}
```

