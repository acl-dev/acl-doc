---
title: acl 日志记录方式介绍
date: 2013-06-23 23:18
categories: 日志
---

在使用 acl 库编写应用过程中，记录日志是一个非常重要的过程，acl 从几个层面提供了日志的不同记录方式。在 acl 的 C 库部分(lib_acl.a)，有三个源文件与日志记录相关：acl_msg.c/acl_msg.h, acl_mylog.c/acl_mylog.h, acl_debug.c/acl_debug.h。其中，acl_mylog.c 是真正记录日志的源文件，acl_msg.c 则是在 acl_mylog.c 基础之上的二次封装，acl_debug.c 是在 acl_msg.c 基础之上的再次封装。下面根据此三个日志源文件从三个层次描述日志记录的过程。

##  一、ac_mylog.c/acl_mylog.h
打开 acl_mylog.h 头文件，可以看到主要有三个函数：acl_open_log（打开日志文件），acl_write_to_log（写日志）以及acl_close_log（关闭日志）---（这三个函数是最基础的日志记录过程，当然我们不必直接使用）。该库支持两类日志记录方式：1、本地文件记录方式，2、与 syslog-ng 结合的网络日志记录方式。本地文件记录方式是 acl 日志库对外提供的最简单的日志记录方式，此方式不依赖于第三方日志库，但不应用在生产环境中，因为该方式不支持日志回滚等高级特性，为了便于生产上使用，所以产生了第二种方式（与 syslog-ng 结合），查看日志打开接口（如下）：

```c++
/**
 * 打开日志文件
 * @param recipients {const char*} 日志接收器列表，由 "|" 分隔，接收器
 *  可以是本地文件或远程套接口，如:
 *  /tmp/test.log|UDP:127.0.0.1:12345|TCP:127.0.0.1:12345|UNIX:/tmp/test.sock
 *  该配置要求将所有日志同时发给 /tmp/test.log, UDP:127.0.0.1:12345,
 *  TCP:127.0.0.1:12345 和 UNIX:/tmp/test.sock 四个日志接收器对象
 * @param plog_pre {const char*} 日志记录信息前的提示信息，建议用进程
 *  名填写此值
 */
ACL_API int acl_open_log(const char *recipients, const char *plog_pre);
```

从上面的函数声明可以看出，acl 的日志记录允许同时输出至多个日志管道中（最简单的方式就是直接写入本地磁盘文件：/tmp/test.log），同时更应看到，其中有三个奇怪的日志文件表达方式：UDP:IP:PORT, TCP:IP:PORT, UNIX:/xxx，其实这三种方式均是与 syslog-ng 相关，即分别表示：

- 1、以 UDP 方式发送日志至 syslog-ng；
- 2、以 TCP 方式发送日志至 syslog-ng；
- 3、以 UNIX 域套接字方式发送日志至 syslog-ng。

因为日志管理是一个非常复杂的过程，所以在 acl 除了提供最简单的日志文件记录外，更建议用户将日志输出至 syslog-ng 中（作者自己的项目也往往是这样做的）。

## 二、acl_msg.c/acl_msg.h
该日志库提供了更为高级的日志记录方法，不仅提供了灵活的日志记录函数，同时还允许用户注册自己的日志记录函数库，该日志库主要函数接口如下：

```c++
/**
 * 日志打开函数
 * @param log_file {const char*} 日志接收者集合，由 "|" 分隔，接收器
 *  可以是本地文件或远程套接口，如:
 *  /tmp/test.log|UDP:127.0.0.1:12345|TCP:127.0.0.1:12345|UNIX:/tmp/test.sock
 *  该配置要求将所有日志同时发给 /tmp/test.log, UDP:127.0.0.1:12345,
 *  TCP:127.0.0.1:12345 和 UNIX:/tmp/test.sock 四个日志接收器对象
 * @param plog_pre {const char*} 日志记录信息前的提示信息，建议用进程
 * @param info_pre {const char*} 日志记录信息前的提示信息
 */
ACL_API void acl_msg_open(const char *log_file, const char *info_pre);

/**
 * 关闭日志函数
 */
ACL_API void acl_msg_close(void);

       上面是日志打开与关闭的函数，看上去算是相对简单。下面是几个日志记录的函数接口：

/**
 * 一般级别日志信息记录函数
 * @param fmt {const char*} 参数格式
 * @param ... 变参序列
 */
#ifdef	WIN32
ACL_API void acl_msg_info(const char *fmt,...);
#else
ACL_API void __attribute__((format(printf,1,2)))
	acl_msg_info(const char *fmt,...);
#endif

/**
 * 警告级别日志信息记录函数
 * @param fmt {const char*} 参数格式
 * @param ... 变参序列
 */
#ifdef	WIN32
ACL_API void acl_msg_warn(const char *fmt,...);
#else
ACL_API void __attribute__((format(printf,1,2)))
	acl_msg_warn(const char *fmt,...);
#endif

/**
 * 错误级别日志信息记录函数
 * @param fmt {const char*} 参数格式
 * @param ... 变参序列
 */
#ifdef	WIN32
ACL_API void acl_msg_error(const char *fmt,...);
#else
ACL_API void __attribute__((format(printf,1,2)))
	acl_msg_error(const char *fmt,...);
#endif

/**
 * 致命级别日志信息记录函数
 * @param fmt {const char*} 参数格式
 * @param ... 变参序列
 */
#ifdef	WIN32
ACL_API void acl_msg_fatal(const char *fmt,...);
#else
ACL_API void __attribute__((format(printf,1,2)))
	acl_msg_fatal(const char *fmt,...);
#endif

/**
 * 恐慌级别日志信息记录函数
 * @param fmt {const char*} 参数格式
 * @param ... 变参序列
 */
#ifdef	WIN32
ACL_API void acl_msg_panic(const char *fmt,...);
#else
ACL_API void __attribute__((format(printf,1,2)))
	acl_msg_panic(const char *fmt,...);
#endif
```

可以看到，这些函数的使用方式与 printf 类似，另外，在 UNIX 下使用 GCC 编译时前面还有一个修饰符：__attribute__((format(printf,m,n)))，这主要是方便 gcc 编译器针对变参进行语法检查（大家应该知道变参是如此方便灵活而又如此容易出错）。

为了方便程序开发过程中的调试，下面的函数当用户未调用 acl_msg_open 打开日志而直接使用 acl_msg_xxx 写日志时，决定是否将日志信息输出至屏幕（这个函数应该在程序初始化时调用）：

```c++
/**
 * 当未调用 acl_msg_open 方式打开日志时，调用了 acl_msg_info/error/fatal/warn
 * 的操作，是否允许信息输出至标准输出屏幕上，通过此函数来设置该开关，该开关
 * 仅影响是否需要将信息输出至终端屏幕而不影响是否输出至文件中
 * @param onoff {int} 非 0 表示允许输出至屏幕
 */
ACL_API void acl_msg_stdout_enable(int onoff);

       前面曾说过，acl 的日志库还允许用户使用自己的日志记录过程，但要求用户必须在程序初始化时注册自己的日志处理函数，如下：

/**
 * 在打开日志前调用此函数注册应用自己的日志打开函数、日志关闭函数、日志记录函数
 * @param open_fn {ACL_MSG_OPEN_FN} 自定义日志打开函数
 * @param close_fn {ACL_MSG_CLOSE_FN} 自定义日志关闭函数
 * @param write_fn {ACL_MSG_WRITE_FN} 自定义日志记录函数
 * @param ctx {void*} 自定义参数
 */
ACL_API void acl_msg_register(ACL_MSG_OPEN_FN open_fn, ACL_MSG_CLOSE_FN close_fn,
        ACL_MSG_WRITE_FN write_fn, void *ctx);
```

调用此函数后，以后的日志记录过程（即当用户调用：acl_msg_xxx 相关过程时）的内容便输出便由用户的日志库控制。

除了以上主要的日志函数接口，在 acl_msg 中还提供了以下几个函数，便于用户知晓程序出错原因：

```c++
/**
 * 获得上次系统调用出错时的错误描述信息，该函数内部采用了线程局部变量，所以是线程
 * 安全的，但使用起来更简单些
 * @return {const char *} 返回错误提示信息 
 */
ACL_API const char *acl_last_serror(void);

/**
 * 获得上次系统调用出错时的错误号
 * @return {int} 错误号
 */
ACL_API int acl_last_error(void);
```

## 三、acl_debug.c/acl_debug.h
该日志函数库是在 acl_msg 之上的再一次封装，该库的思想来源于 squid 的日志记录方式，可以将日志分成不同的类别，每一个类别又分成不同的级别，这样用户就可以非常方便地通过配置文件来记录不同类别的不同级别的日志信息了。在程序初始化时需先调用如此函数：

```c++
/**
 * 初始化日志调试调用接口
 * @param pStr {const char*} 调试类别（建议值在100至1000之间）标签及级别字符串，
 *  格式: 1,1; 2,10; 3,8...  or 1:1; 2:10; 3:8...
 */
ACL_API void acl_debug_init(const char *pStr);

/**
 * 初始化日志调试调用接口
 * @param pStr {const char*} 调试标签及级别字符串，
 *  格式: 1,1; 2,10; 3,8...  or 1:1; 2:10; 3:8...
 * @param max_debug_level {int} 最大调试标签值
 */
ACL_API void acl_debug_init2(const char *pStr, int max_debug_level);
```

其中，第一个参数是一个由日志记录类别与级别组成的字符串，格式为：类别1:最大记录级别, 类别2:最大记录级别, ...。例如：100:2; 102:3; 103:4，其含义是日志将会记录类别为 100 的所有级别值小于2、类别为 101 的所有级别值小于 3 以及类别为 103 的所有级别值小于 4 的日志信息。关于记录类别需要注意：类别值最好是 >= 100，且 < 1000（当使用 acl_debug_init2 初始化时只要类别值 >= 100 即可，因为第二个参数指定了最大类别值），这是因为 acl 库内部一些保留的类别值都在 0 -- 100 之间。

那么具体的使用这些类别与级别记录日志的接口是什么呢？如下所示：

```c++
/**
 * 日志调试宏接口
 * @param SECTION {int} 调试标签值
 * @param LEVEL {int} 对应于SECTION调试标签的级别
 */
#define acl_debug(SECTION, LEVEL) \
	!acl_do_debug((SECTION), (LEVEL)) ? (void) 0 : acl_msg_info

        看到了吧，用户其实只需要调用一个宏即可，如下面的例子： 

	/* 初始化日志类别记录 */
	const char *str = "101:2; 103:4; 105:3";
	/* 记录所有类别值为 101 级别小于等于 2、类别值为 102 级别小于等于 4、类别值为 105 级别小于等于 3 的日志内容 */
	acl_debug_init(str);

	......
	/* 下面的日志因符合类别值 101 级别值 <= 2 而被记录 */
	acl_debug(101, 2)("%s(%d): log time: %ld", __FILE__, __LINE__, time(NULL));

	/* 下面日志符合类别 105 的记录级别 */
	acl_debug(105, 1)("%s(%d): log time: %ld", __FILE__, __LINE__, time(NULL));

	/* 下面的日志因不符合类别值 103 的记录级别条件而被忽略 */
	acl_debug(103, 5)("%s(%d): log time: %ld", __FILE__, __LINE__, time(NULL));

	/* 下面日志的类别值 102 因不存在而被忽略 */
	acl_debug(102, 1)("%s(%d): log time: %ld", __FILE__, __LINE__, time(NULL));
```

此外，为了方便，还可以传给 acl_debug_init 的参数写为："all:1"，意思是所有类别的级别值 <= 1 的日志都将被记录，如下面的内容都会被记录：

```
	acl_debug_init("all:1");
......
	acl_debug(100, 1)("%s(%d): log time: %ld", __FILE__, __LINE__, time(NULL));
	acl_debug(101, 1)("%s(%d): log time: %ld", __FILE__, __LINE__, time(NULL));
	acl_debug(101, 0)("%s(%d): log time: %ld", __FILE__, __LINE__, time(NULL));
	acl_debug(102, 1)("%s(%d): log time: %ld", __FILE__, __LINE__, time(NULL));
	acl_debug(103, 1)("%s(%d): log time: %ld", __FILE__, __LINE__, time(NULL));
	acl_debug(104, 1)("%s(%d): log time: %ld", __FILE__, __LINE__, time(NULL));
	acl_debug(105, 1)("%s(%d): log time: %ld", __FILE__, __LINE__, time(NULL));
        ......
```

ok，有关日志  acl 日志记录函数就先写这些，使用者可以根据项目需要采用不同的日志记录方式。

github 地址：https://github.com/acl-dev/acl
gitee 地址：https://gitee.com/acl-dev/acl
 

