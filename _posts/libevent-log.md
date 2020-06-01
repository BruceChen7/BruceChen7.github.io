title: libevent源码分析之log
date: 2015-01-10 14:56:42
tags:  [Libevent,C]
---


# log.c源码分析

# 一些定义

* typedef void (*event_fata_cb)(int err); 是一个函数指针,是一个指向回调函数．也就是发生错误的时候，调用的回调函数.
* static event_fatal_cb fatal_fn = NULL;
* `#define ev_uint32_t uint32_t` 指定数据类型
* static ev_uint32_t event_debug_logging_mask_ =  DEFAULT_MASK_;，指定debug级别,在这个文件中是全局的静态变量
* typedef void(*event_log_cb)(int severity,const char *msg);
* static event_log_cb  log_fn = NULL;

<!--more-->

## 本文件中要用到的函数

	static void event_log(int severity,const char *)

	//如果有自定义自己的发生错误时回调函数,则调用，
	//如果errcode是EVENT_ERR_ABOART_;
	//则调用abort()终止程序
	//否则调用exit
	static void event_exit(int errcode) EV_NORETURN
	//默认设置发生错误时的回调函数
	static event_fatal_cb fatal_fn = NULL; //这是一个适用于本文件的全局的变量
	static event_log(int serverity,const char *msg);


# 对外(也就是先项目本身这里指的不是使用libevent的用户)能够调用的函数

* ev_uint32_t event_get_logging_mask_(void);
* void event_set_fatal_callback(event_fatal_cb cb); //设置发生fatal internal error时调用的函数
* void event_err(int eval,const char *fmt,...);
* void event_warn(const char *fmt);
* void event_sock_err(int eval,evutil_socket_t sock,const char *fmt,...);
* void event_sock_warn(evutil_socket_t sock,const char *fmt,....);
* void event_warnx(const char *fmt);
* void event_errx(int eval,const char *fmt,...);
* void event_msgx(const char *fmt,...);
* void event_debugx_(const char *fmt,...)
* void event_logv_(int severity,const char *errstr,const char *fmt,va_list ap)

## 重点代码解析
在上面列出的函数中，比较重要的函数是`event_logv_`这个函数，`event_err`,`event_warn`,`event_errx`,`event_msgx`等都直接调用调用了这个函数，所以这些函数只是`wrapper`.现在来看看实现的代码:

	void event_logv_(int severity, const char *errstr, const char *fmt, va_list ap)
	{
		char buf[1024];
		size_t len;

		if (severity == EVENT_LOG_DEBUG && !event_debug_get_logging_mask_())
			return;

		if (fmt != NULL)
			evutil_vsnprintf(buf, sizeof(buf), fmt, ap);
		else
			buf[0] = '\0';

		if (errstr) {
			len = strlen(buf);
			if (len < sizeof(buf) - 3) {
				evutil_snprintf(buf + len, sizeof(buf) - len, ": %s", errstr);
			}
		}

		event_log(severity, buf);
	}

首先聊一聊`severity`,libevent支持的日志级别为４类:

* EVENT_LOG_DEBUG  0
* EVENT_LOG_MSG    1
* EVENT_LOG_WARN   2
* EVENT_LOG_ERR    3

这个函数首先判定日志的`severity`的级别，如果日志的级别是`EVENT_LOG_DEBUG`也就是调试级别，并且没有打开DEBUG功能，那么什么都不做．注意到`event_debug_get_logging_mask()`返回的是`event_debug_loggin_mask_`这是一个`static uint32_t`值，在没有`#define EVENT_DEBUG_LOGGING_ENABLE`和`＃define USE_DEBUG`的条件下，默认值为０，定义了，这个值为`#ffffffffu`

第二个判别是根据是否定义了格式化字符比如`%s,%d`等，如果有定义了，则调用`evutil_vsnprintf()`函数，否则将`buf`中置为空．

再有输入错误信息的情况下,判别当前buf空余的空间是否大于３个字节，如果大于则在`buf+len`处写入`sizeof(buf) - len`大小的错误信息.

* 最后通过调用用`event_log`，将格式化的buf和日志级别写入到终端.

## 总结一下`event_logv_`干的事情

* 如果没有打开`debug`，并且当前的日志级别为`debug`什么都不做．
* 其它的情况下，我们首先格式化buf,然后将其传给`event_log`显示到终端．


## 聊一聊event_log函数
这个函数主要由`event_logv_`来调用，注意其声明为`static`.


```cpp
static void event_log(int severity, const char *msg)
{
    if (log_fn)
        log_fn(severity, msg);
    else {
        const char *severity_str;
        switch (severity) {
        case EVENT_LOG_DEBUG:
            severity_str = "debug";
            break;
        case EVENT_LOG_MSG:
            severity_str = "msg";
            break;
        case EVENT_LOG_WARN:
            severity_str = "warn";
            break;
        case EVENT_LOG_ERR:
            severity_str = "err";
            break;
        default:
            severity_str = "???";
            break;
        }
        (void)fprintf(stderr, "[%s] %s\n", severity_str, msg);
    }
}
```
这个函数根据日志的级别，显示不同的日志级别信息，没什么很特别．

# 总结

* 总的来说，`libevent`的日志没有什么很特别的东西，它将日志信息输出到终端，不是很好，不利于日后查看．
* 有什么可以学习的，我认为是其对于可变函数的使用，以前没有接触过．现在知道了其作用．对于va_list的使用，可以看[这篇](http://brucechen.gitcafe.com/C/Functions/va_list.html)
