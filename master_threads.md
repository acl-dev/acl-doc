---
title: 使用 acl::master_threads 类编写多进程多线程服务器程序
date: 2012-05-26 13:02
categories: 服务编程
---

本文主要讲述如何使用 acl_cpp 中的 master_threads 类编写可以由 acl_master 服务器父进程控制的服务器应用程序。

## 一、类接口说明
master_threads 是一个纯虚类，其中定义的接口需要子类实现，如下：
```c++
	/**
	 * 纯虚函数：当某个客户端连接有数据可读或关闭或出错时调用此函数
	 * @param stream {socket_stream*}
	 * @return {bool} 返回 false 则表示当函数返回后需要关闭连接，
	 *  否则表示需要保持长连接，如果该流出错，则应用应该返回 false
	 */
	virtual bool thread_on_read(socket_stream*) = 0;

	/**
	 * 当线程池中的某个线程获得一个连接时的回调函数，
	 * 子类可以做一些初始化工作
	 * @param stream {socket_stream*}
	 * @return {bool} 如果返回 false 则表示子类要求关闭连接，而不
	 *  必将该连接再传递至 thread_main 过程
	 */
	virtual bool thread_on_accept(socket_stream*) { return true; }

	/**
	 * 当与某个线程绑定的连接关闭时的回调函数
	 * @param stream {socket_stream*}
	 */
	virtual void thread_on_close(socket_stream* ) {}

	/**
	 * 当线程池中一个新线程被创建时的回调函数
	 */
	virtual void thread_on_init() {}

	/**
	 * 当线程池中一个线程退出时的回调函数
	 */
	virtual void thread_on_exit() {}
```

master_threads 类提供了两个函数：
```c++
	/**
	 * 开始运行，调用该函数是指该服务进程是在 acl_master 服务框架
	 * 控制之下运行，一般用于生产机状态
	 * @param argc {int} 从 main 中传递的第一个参数，表示参数个数
	 * @param argv {char**} 从 main 中传递的第二个参数
	 */
	void run_daemon(int argc, char** argv);

	/**
	 * 在单独运行时的处理函数，用户可以调用此函数进行一些必要的调试工作
	 * @param addr {const char*} 服务监听地址
	 * @param conf {const char*} 配置文件全路径
	 * @param count {unsigned int} 循环服务的次数，达到此值后函数自动返回；
	 *  若该值为 0 则表示程序一直循环处理外来请求而不返回
	 * @param threads_count {int} 当该值大于 1 时表示自动采用线程池方式，
	 *  该值只有当 count != 1 时才有效，即若 count == 1 则仅运行一次就返回
	 *  且不会启动线程处理客户端请求
	 * @return {bool} 监听是否成功
	 */
	bool run_alone(const char* addr, const char* conf = NULL,
		unsigned int count = 1, int threads_count = 1);
```

从上面两个函数，可以看出 master_threads 类当在生产环境下（由 acl_master 进程统一控制调度），用户需要调用 run_daemon 函数；如果用户在开发过程中需要手工进行调试，则可以调用 run_alone 函数。

master_threads 的基类 master_base 的几个虚接口如下：
```c++
	/**
	 * 当进程切换用户身份前调用的回调函数，可以在此函数中做一些
	 * 用户身份为 root 的权限操作
	 */
	virtual void proc_pre_jail() {}

	/**
	 * 当进程切换用户身份后调用的回调函数，此函数被调用时，进程
	 * 的权限为普通受限级别
	 */
	virtual void proc_on_init() {}

	/**
	 * 当进程退出前调用的回调函数
	 */
	virtual void proc_on_exit() {}
```

	// 在 run_alone 状态下运行前，调用此函数初始化一些配置

基类的这几个虚函数用户可以根据需要调用。
 
另外，基类 master_base 还提供了几个用来读配置选项的函数：
```c++
	/**
	 * 设置 bool 类型的配置项
	 * @param table {master_bool_tbl*}
	 */
	void set_cfg_bool(master_bool_tbl* table);

	/**
	 * 设置 int 类型的配置项
	 * @param table {master_int_tbl*}
	 */
	void set_cfg_int(master_int_tbl* table);

	/**
	 * 设置 int64 类型的配置项
	 * @param table {master_int64_tbl*}
	 */
	void set_cfg_int64(master_int64_tbl* table);

	/**
	 * 设置 字符串 类型的配置项
	 * @param table {master_str_tbl*}
	 */
	void set_cfg_str(master_str_tbl* table);
```

## 二、示例源程序

```c++
// master_threads.cpp : 定义控制台应用程序的入口点。
//

#include "log.hpp"
#include "util.hpp"
#include "master_threads.hpp"
#include "socket_stream.hpp"

// 字符串类型的配置项
static char *var_cfg_debug_msg;

static acl::master_str_tbl var_conf_str_tab[] = {
	{ "debug_msg", "test_msg", &var_cfg_debug_msg },

	{ 0, 0, 0 }
};

// 布尔类型的配置项
static int  var_cfg_debug_enable;
static int  var_cfg_keep_alive;
static int  var_cfg_loop;

static acl::master_bool_tbl var_conf_bool_tab[] = {
	{ "debug_enable", 1, &var_cfg_debug_enable },
	{ "keep_alive", 1, &var_cfg_keep_alive },
	{ "loop_read", 1, &var_cfg_loop },

	{ 0, 0, 0 }
};

// 整数类型的配置项
static int  var_cfg_io_timeout;

static acl::master_int_tbl var_conf_int_tab[] = {
	{ "io_timeout", 120, &var_cfg_io_timeout, 0, 0 },

	{ 0, 0 , 0 , 0, 0 }
};

static void (*format)(const char*, ...) = acl::log::msg1;

//////////////////////////////////////////////////////////////////////////

class master_threads_test : public acl::master_threads
{
public:
	master_threads_test()
	{
	}

	~master_threads_test()
	{
	}

protected:
	// 基类纯虚函数：当客户端连接有数据可读或关闭时回调此函数，返回 true 表示
	// 继续与客户端保持长连接，否则表示需要关闭客户端连接
	bool thread_on_read(acl::socket_stream* stream)
	{
		while (true)
		{
			if (on_read(stream) == false)
				return false;
			if (var_cfg_loop == 0)
				break;
		}
		return true;
	}

	bool on_read(acl::socket_stream* stream)
	{
		format("%s(%d)", __FILE__, __LINE__);
		acl::string buf;
		if (stream->gets(buf) == false) // 读一行数据
		{
			format("gets error: %s", acl::last_serror());
			format("%s(%d)", __FILE__, __LINE__);
			return false;
		}
		if (buf == "quit")  // 如果客户端要求关闭连接，则返回 false 通知服务器框架关闭连接
		{
			stream->puts("bye!");
			return false;
		}

		if (buf.empty())
		{
			if (stream->write("\r\n") == -1)
			{
				format("write 1 error: %s", acl::last_serror());
				return false;
			}
		}
		else if (stream->write(buf) == -1)
		{
			format("write 2 error: %s, buf(%s), len: %d",
				acl::last_serror(), buf.c_str(), (int) buf.length());
			return false;
		}
		else if (stream->write("\r\n") == -1)
		{
			format("write 3 client error: %s", acl::last_serror());
			return false;
		}
		return true;
	}

	// 基类虚函数：当接收到一个客户端请求时，调用此函数，允许
	// 子类事先对客户端连接进行处理，返回 true 表示继续，否则
	// 要求关闭该客户端连接
	bool thread_on_accept(acl::socket_stream* stream)
	{
		format("accept one client, peer: %s, local: %s, var_cfg_io_timeout: %d\r\n",
			stream->get_peer(), stream->get_local(), var_cfg_io_timeout);
		if (stream->format("hello, you're welcome!\r\n") == -1)
			return false;
		return true;
	}

	// 基类虚函数：当客户端连接关闭时调用此函数
	void thread_on_close(acl::socket_stream*)
	{
		format("client closed now\r\n");
	}

	// 基类虚函数：当线程池创建一个新线程时调用此函数
	void thread_on_init()
	{
#ifdef WIN32
		format("thread init: tid: %lu\r\n", GetCurrentThreadId());
#else
		format("thread init: tid: %lu\r\n", pthread_self());
#endif
	}

	// 基类虚函数：当线程池中的一个线程退出时调用此函数
	virtual void thread_on_exit()
	{
#ifdef WIN32
		format("thread exit: tid: %lu\r\n", GetCurrentThreadId());
#else
		format("thread exit: tid: %lu\r\n", pthread_self());
#endif
	}

	// 基类虚函数：服务进程切换用户身份前调用此函数
	void proc_pre_jail()
	{
		format("proc_pre_jail\r\n");
	}

	// 基类虚函数：服务进程切换用户身份后调用此函数
	void proc_on_init()
	{
		format("proc init\r\n");
	}

	// 基类虚函数：服务进程退出前调用此函数
	void proc_on_exit()
	{
		format("proc exit\r\n");
	}
};

int main(int argc, char* argv[])
{
	master_threads_test mt;

	// 设置配置参数表
	mt.set_cfg_int(var_conf_int_tab);
	mt.set_cfg_int64(NULL);
	mt.set_cfg_str(var_conf_str_tab);
	mt.set_cfg_bool(var_conf_bool_tab);

	// 开始运行

	if (argc >= 2 && strcmp(argv[1], "alone") == 0)
	{
		format = (void (*)(const char*, ...)) printf;
		format("listen: 127.0.0.1:8888\r\n");
		mt.run_alone("127.0.0.1:8888", NULL, 2, 10);  // 单独运行方式
	}
	else
		mt.run_daemon(argc, argv);  // acl_master 控制模式运行
	return 0;
}
```

这是一个简单的提供 echo 行服务的服务器程序，可以支持多个并发连接，而且可以通过配置文件控制所启动的最大进程数、每个进程的最大线程数、进程空闲时间、线程空闲时间等控制参数，因为 acl 中的服务器框架都是半驻留的，所以既可以保证运行效率，又能够在空闲释放系统资源。该例子所在目录：acl_cpp/samples/master_threads。

需要指出的一点是，master_threads 内部是单例的，即要求该类的对象只能有一个，否则内部自动产生断言。只所以没有采用单例模板来设计单例，主要是为了不对外暴露 acl 库中的接口，使使用 acl_cpp 库的用户不必关心 acl 库的头文件在哪儿。

## 三、配置文件及程序安装

打开 acl_cpp/samples/master_threads/ioctl_echo.cf 配置文件，就其中几个配置参数说明一下：

```
## 由 acl_master 用来控制服务进程池的配置项
# 为 no 表示启用该进程服务，为 yes 表示禁止该服务进程
master_disable = no

# 表示本服务器进程监听 127.0.0.1 的 5001 端口
master_service = 127.0.0.1:5001

# 表示是 TCP 套接口服务类型
master_type = inet

# 表示该服务进程池的最大进程数为 2
master_maxproc = 2

# 进程日志记录文件，其中 {install_path} 需要用实际的安装路径代替
master_log = {install_path}/var/log/ioctl_echo.log

## 与该服务器框架模板相关的配置参数项
# 每个服务进程中最大的线程数为 250
ioctl_max_threads = 250

# 线程的堆栈空间大小，单位为字节，0表示使用系统缺省值
ioctl_stacksize = 0

# 每个进程实例处理连接数的最大次数，超过此值后进程实例主动退出
ioctl_use_limit = 100

# 每个进程实例的空闲超时时间，超过此值后进程实例主动退出
ioctl_idle_limit = 120

# 进程运行时的用户身份
ioctl_owner = root

# 采用事件循环的方式: select(default)/poll/kernel(epoll/devpoll/kqueue)
ioctl_event_mode = select

# 允许访问 udserver 的客户端IP地址范围
ioctl_access_allow = 10.0.0.1:10.0.0.255, 127.0.0.1:127.0.0.1
```

例如当 acl_master 服务器框架程序的安装目录为：/opt/acl，则：

- /opt/acl/libexec： 该目录存储服务器程序（acl_master 程序也存放在该目录下）；
- /opt/acl/conf：该目录存放 acl_master 程序配置文件 main.cf；
- /opt/acl/conf/service：该目录存放服务子进程的程序配置文件，该路径由 main.cf 文件指定；
- /opt/acl/var/log：该目录存放日志文件；
- /opt/acl/var/pid：该目录存放进程号文件。

该程序编译通过后，需要把可执行程序放在 /opt/acl/libexec 目录下，把配置文件放在 /opt/acl/conf/service 目录下。

在 /opt/acl/sh 下有启动/停止 acl_master 服务进程的控制脚本；运行脚本：./start.sh，然后请用下面方法检查服务是否已经启动：

`$ps -ef|grep acl_master`  查看服务器控制进程是否已经启动

`$netstat -nap|grep LISTEN|grep 5001` 查看服务端口号是否已经被监听

当然，您也可以查看 /opt/acl/var/log/acl_master 日志文件，查看服务进程的启动过程及监听服务是否正常监听。

可以命令行如下测试：`$telnet 127.0.0.1 5001`