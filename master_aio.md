---
title: 用 acl::master_aio 类编写高并发非阻塞服务器程序
date: 2012-05-30 22:15
categories: 服务编程
---

在文章《使用 acl::master_threads 类编写多进程多线程服务器程序》讲述了如何编写 LINUX 平台下阻塞式服务器程序的多线程。虽然这种模式都可以处理并发任务，并且效率也不低，但是毕竟线程和进程资源是操作系统的宝贵资源，如果要支持非常高的并发请求，则会因为系统限制而不能创建更多的进程或线程。大家常见的 webserver Nginx 就是以支持高并发而闻名，Nginx 本身就是非阻塞设计的典型应用，当然还有很多其它服务器程序也是非阻塞的：ircd、Squid、Varnish、Bind 等。

因为非阻塞程序编写有一定难度，所以现在有人在写非阻塞程序时不得不转向 Erlang、Nodejs 等一些脚本式语言，要想用 C/C++ 来实现非阻塞程序还是有些难度：系统级 API 过于底层、容易出错、难于调试。虽然也有一些 C++ 库（象ACE）提供了非阻塞库，但这些库的入门门槛还是挺高的。根据本人多年在非阻塞编程方面的经验，总结出一套比较实用的 C/C++ 非阻塞函数库，本文就讲述如果使用 acl_cpp 库中的 master_aio 类来编写非阻塞服务器程序（该服务器程序可以规定启动最大进程的个数，其中每个进程是一个独立的非阻塞进程）。在文章《非阻塞网络编程实例讲解》中给出了一个非阻塞网络程序的示例，您可以参考一下。

## 一、类接口定义
master_aio 是一个纯虚类，其中定义的接口需要子类实现，子类实例必须只能创建一个实例，接口如下：
```c++
	/**
	 * 纯虚函数：当接收到一个客户端连接时调用此函数
	 * @param stream {aio_socket_stream*} 新接收到的客户端异步流对象
	 * @return {bool} 该函数如果返回 false 则通知服务器框架不再接收
	 *  远程客户端连接，否则继续接收客户端连接
	 */
	virtual bool on_accept(aio_socket_stream* stream) = 0;
```

master_aio 提供了四个外部方法，如下：
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
	 * @param ht {aio_handle_type} 事件引擎的类型
	 * @return {bool} 监听是否成功
	 */
	bool run_alone(const char* addr, const char* conf = NULL,
		aio_handle_type ht = ENGINE_SELECT);

	/**
	 * 获得异步IO的事件引擎句柄，通过此句柄，用户可以设置定时器等功能
	 * @return {aio_handle*}
	 */
	aio_handle* get_handle() const;

	/**
	 * 在 run_alone 模式下，通知服务器框架关闭引擎，退出程序
	 */
	void stop();
```

从上面两个函数，可以看出 master_aio 类当在生产环境下（由 acl_master 进程统一控制调度），用户需要调用 run_daemon 函数；如果用户在开发过程中需要手工进行调试，则可以调用 run_alone 函数。

master_aio 的基类 master_base 的几个虚接口如下：
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

	// 在 run_alone 状态下运行前，调用此函数初始化一些配置
```

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
// master_aio.cpp : 定义控制台应用程序的入口点。
//

#include "stdafx.h"
#include "lib_acl.hpp"

static char *var_cfg_debug_msg;

static acl::master_str_tbl var_conf_str_tab[] = {
	{ "debug_msg", "test_msg", &var_cfg_debug_msg },

	{ 0, 0, 0 }
};

static int  var_cfg_debug_enable;
static int  var_cfg_keep_alive;
static int  var_cfg_send_banner;

static acl::master_bool_tbl var_conf_bool_tab[] = {
	{ "debug_enable", 1, &var_cfg_debug_enable },
	{ "keep_alive", 1, &var_cfg_keep_alive },
	{ "send_banner", 1, &var_cfg_send_banner },

	{ 0, 0, 0 }
};

static int  var_cfg_io_timeout;

static acl::master_int_tbl var_conf_int_tab[] = {
	{ "io_timeout", 120, &var_cfg_io_timeout, 0, 0 },

	{ 0, 0 , 0 , 0, 0 }
};

static void (*format)(const char*, ...) = acl::log::msg1;

using namespace acl;

//////////////////////////////////////////////////////////////////////////
/**
 * 延迟读回调处理类
 */
class timer_reader: public aio_timer_reader
{
public:
	timer_reader(int delay)
	{
		delay_ = delay;
		format("timer_reader init, delay: %d\r\n", delay);
	}

	~timer_reader()
	{
	}

	// aio_timer_reader 的子类必须重载 destroy 方法
	void destroy()
	{
		format("timer_reader delete, delay: %d\r\n", delay_);
		delete this;
	}

	// 重载基类回调方法
	void timer_callback(unsigned int id)
	{
		format("timer_reader(%u): timer_callback, delay: %d\r\n", id, delay_);

		// 调用基类的处理过程
		aio_timer_reader::timer_callback(id);
	}

private:
	int   delay_;
};

/**
 * 延迟写回调处理类
 */
class timer_writer: public aio_timer_writer
{
public:
	timer_writer(int delay)
	{
		delay_ = delay;
		format("timer_writer init, delay: %d\r\n", delay);
	}

	~timer_writer()
	{
	}

	// aio_timer_reader 的子类必须重载 destroy 方法
	void destroy()
	{
		format("timer_writer delete, delay: %d\r\n", delay_);
		delete this;
	}

	// 重载基类回调方法
	void timer_callback(unsigned int id)
	{
		format("timer_writer(%u): timer_callback, delay: %u\r\n", id, delay_);

		// 调用基类的处理过程
		aio_timer_writer::timer_callback(id);
	}

private:
	int   delay_;
};

class timer_test : public aio_timer_callback
{
public:
	timer_test() {}
	~timer_test() {}
protected:
	// 基类纯虚函数
	void timer_callback(unsigned int id)
	{
		format("id: %u\r\n", id);
	}

	void destroy(void)
	{
		delete this;
		format("timer delete now\r\n");
	}
};

/**
 * 异步客户端流的回调类的子类
 */
class io_callback : public aio_callback
{
public:
	io_callback(aio_socket_stream* client)
		: client_(client)
		, i_(0)
	{
	}

	~io_callback()
	{
		format("delete io_callback now ...\r\n");
	}

	/**
	 * 实现父类中的虚函数，客户端流的读成功回调过程
	 * @param data {char*} 读到的数据地址
	 * @param len {int} 读到的数据长度
	 * @return {bool} 返回 true 表示继续，否则希望关闭该异步流
	 */
	bool read_callback(char* data, int len)
	{
		if (++i_ < 10)
			format(">>gets(i: %d): %s\r\n", i_, data);

		// 如果远程客户端希望退出，则关闭之
		if (strncmp(data, "quit", 4) == 0)
		{
			client_->format("Bye!\r\n");
			client_->close();
		}

		// 如果远程客户端希望服务端也关闭，则中止异步事件过程
		else if (strncmp(data, "stop", 4) == 0)
		{
			client_->format("Stop now!\r\n");
			client_->close();  // 关闭远程异步流

			// 通知异步引擎关闭循环过程
			client_->get_handle().stop();
		}

		// 向远程客户端回写收到的数据

		int   delay = 0;

		if (strncmp(data, "write_delay", strlen("write_delay")) == 0)
		{
			// 延迟写过程

			const char* ptr = data + strlen("write_delay");
			delay = atoi(ptr);
			if (delay > 0)
			{
				format(">> write delay %d second ...\r\n", delay);
				timer_writer* timer = new timer_writer(delay);
				client_->write(data, len, delay * 1000000, timer);
				client_->gets(10, false);
				return true;
			}
		}
		else if (strncmp(data, "read_delay", strlen("read_delay")) == 0)
		{
			// 延迟读过程

			const char* ptr = data + strlen("read_delay");
			delay = atoi(ptr);
			if (delay > 0)
			{
				client_->write(data, len);
				format(">> read delay %d second ...\r\n", delay);
				timer_reader* timer = new timer_reader(delay);
				client_->gets(10, false, delay * 1000000, timer);
				return true;
			}
		}

		client_->write(data, len);
		//client_->gets(10, false);
		return true;
	}

	/**
	 * 实现父类中的虚函数，客户端流的写成功回调过程
	 * @return {bool} 返回 true 表示继续，否则希望关闭该异步流
	 */
	bool write_callback()
	{
		return true;
	}

	/**
	 * 实现父类中的虚函数，客户端流的关闭回调过程
	 */
	void close_callback()
	{
		// 必须在此处删除该动态分配的回调类对象以防止内存泄露
		delete this;
	}

	/**
	 * 实现父类中的虚函数，客户端流的超时回调过程
	 * @return {bool} 返回 true 表示继续，否则希望关闭该异步流
	 */
	bool timeout_callback()
	{
		format("Timeout ...\r\n");
		return true;
	}

private:
	aio_socket_stream* client_;
	int   i_;
};

//////////////////////////////////////////////////////////////////////////

class master_aio_test : public master_aio
{
public:
	master_aio_test() { timer_test_ = new timer_test(); }

	~master_aio_test() { handle_->keep_timer(false); }

protected:
	// 基类纯虚函数：当接收到一个新的连接时调用此函数
	bool on_accept(aio_socket_stream* client)
	{
		// 创建异步客户端流的回调对象并与该异步流进行绑定
		io_callback* callback = new io_callback(client);

		// 注册异步流的读回调过程
		client->add_read_callback(callback);

		// 注册异步流的写回调过程
		client->add_write_callback(callback);

		// 注册异步流的关闭回调过程
		client->add_close_callback(callback);

		// 注册异步流的超时回调过程
		client->add_timeout_callback(callback);

		// 写欢迎信息
		if (var_cfg_send_banner)
			client->format("hello, you're welcome\r\n");

		// 从异步流读一行数据
		client->gets(10, false);
		//client->read();
		return true;
	}

	// 基类虚函数：服务进程切换用户身份前调用此函数
	void proc_pre_jail()
	{
		format("proc_pre_jail\r\n");
		// 只有当程序启动后才能获得异步引擎句柄
		handle_ = get_handle();
		handle_->keep_timer(true); // 允许定时器被重复触发
		// 设置第一个定时任务，每隔1秒触发一次，定时任务ID为0
		handle_->set_timer(timer_test_, 1000000, 0);
	}

	// 基类虚函数：服务进程切换用户身份后调用此函数
	void proc_on_init()
	{
		format("proc init\r\n");
		// 设置第二个定时任务，每隔2秒触发一次，定时任务ID为1
		handle_->set_timer(timer_test_, 2000000, 1);
	}

	// 基类虚函数：服务进程退出前调用此函数
	void proc_on_exit()
	{
		format("proc exit\r\n");
	}
private:
	timer_test* timer_test_;
	aio_handle* handle_;
};
//////////////////////////////////////////////////////////////////////////

int main(int argc, char* argv[])
{
	master_aio_test ma;  // master_aio 要求只能起一个实例

	// 设置配置参数表
	ma.set_cfg_int(var_conf_int_tab);
	ma.set_cfg_int64(NULL);
	ma.set_cfg_str(var_conf_str_tab);
	ma.set_cfg_bool(var_conf_bool_tab);

	// 开始运行

	if (argc >= 2 && strcmp(argv[1], "alone") == 0)
	{
		const char* addr = "127.0.0.1:8888";

		if (argc >= 3)
			addr = argv[2];

		format = (void (*)(const char*, ...)) printf;
		format("listen: %s now\r\n", addr);
		ma.run_alone(addr);  // 单独运行方式
	}
	else
		ma.run_daemon(argc, argv);  // acl_master 控制模式运行
	return 0;
}
```

该示例主要实现几个功能：接收一行数据并回写该数据、延迟回写所读的数据、延迟读下一行数据、设置定时器的定时任务。因为 master_aio 类是按非阻塞模式设计的（其实，该类主要对 acl 库中的非阻塞框架库用 C++ 进行了封装），所以该例子可以支持非常大的并发请求。可以通过配置文件指定系统事件 API 选择采用 select/poll/epoll 中的一种、规定进程的空闲退出时间、预先启动子进程的数量等参数。该例子所在目录：acl_cpp/samples/master_aio。

在这个例子中，当服务端接收到客户端连接后，非阻塞框架会通过虚函数  on_accept 回调子类处理过程，子类需要在这个虚函数中需要将一个处理 IO 过程的类实例与这个非阻塞连接流进行绑定（通过 add_xxx_callback 方式），其中处理 IO 过程的类的基类定义为：aio_callback，子类需要实现其中的虚方法。

## 三、配置文件及程序安装
打开 acl_cpp/samples/master_aio/aio_echo.cf 配置文件，就其中几个配置参数说明一下：

```
## 由 acl_master 用来控制服务进程池的配置项
# 为 no 表示启用该进程服务，为 yes 表示禁止该服务进程
master_disable = no

# 表示本服务器进程监听 127.0.0.1 的 8888 端口
master_service = 127.0.0.1:8888

# 表示是 TCP 套接口服务类型
master_type = inet

# 表示该服务进程池的最大进程数为 2
master_maxproc = 2

# 需要预先启动的进程数，该值不应大于 master_maxproc
master_prefork = 2

# 进程程序名
master_command = aio_echo

# 进程日志记录文件，其中 {install_path} 需要用实际的安装路径代替
master_log = {install_path}/var/log/aio_echo.log
 
# 用 select 进行循环时的时间间隔
# 单位为秒
aio_delay_sec = 1
# 单位为微秒
aio_delay_usec = 500
# 采用事件循环的方式: select(default), poll, kernel(epoll/devpoll/kqueue)
aio_event_mode = select
# 是否将 socket 接收与IO功能分开: yes/no, 如果为 yes 可以大大提高 accept() 速度
aio_accept_alone = yes
# 线程池的最大线程数, 如果该值为0则表示采用单线程非阻塞模式.
aio_max_threads = 0
# 每个线程的空闲时间.
aio_thread_idle_limit = 60
# 允许访问的客户端IP地址范围
aio_access_allow = 10.0.0.1:255.255.255.255, 127.0.0.1:127.0.0.1
# 当 acl_master 退出时，如果该值置1则该程序不等所有连接处理完毕便立即退出
aio_quick_abort = 1
```

例如当 acl_master 服务器框架程序的安装目录为：/opt/acl，则：

- /opt/acl/libexec： 该目录存储服务器程序（acl_master 程序也存放在该目录下）；
- /opt/acl/conf：该目录存放 acl_master 程序配置文件 main.cf；
- /opt/acl/conf/service：该目录存放服务子进程的程序配置文件，该路径由 main.cf 文件指定；
- /opt/acl/var/log：该目录存放日志文件；
- /opt/acl/var/pid：该目录存放进程号文件。

该程序编译通过后，需要把可执行程序放在 /opt/acl/libexec 目录下，把配置文件放在 /opt/acl/conf/service 目录下。

在 /opt/acl/sh 下有启动/停止 acl_master 服务进程的控制脚本；运行脚本：./start.sh，然后请用下面方法检查服务是否已经启动：

ps -ef|grep acl_master # 查看服务器控制进程是否已经启动

netstat -nap|grep LISTEN|grep 5001 # 查看服务端口号是否已经被监听

当然，您也可以查看 /opt/acl/var/log/acl_master 日志文件，查看服务进程的启动过程及监听服务是否正常监听。

可以命令行如下测试：telnet 127.0.0.1 8888

github: https://github.com/acl-dev/acl
gitee: https://gitee.com/acl-dev/acl
 
