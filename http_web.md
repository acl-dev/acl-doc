---
title: 使用 acl_cpp 的 HttpServlet 类及服务器框架编写WEB服务器程序
date: 2012-05-21 13:08:24
tags: http
categories: http开发
---

在 《用C++实现类似于JAVA HttpServlet 的编程接口 》 文章中讲了如何用 HttpServlet 等相关类编写 CGI 程序，于是有网友提出了 CGI 程序低效性，不错，确实 CGI 程序的进程开销是比较大的，本文就将说明依然是这些 HTTP 相关的类，如果在使用 acl_cpp/src/master 下的服务器框架类的条件下，可以非常方便地转为服务器程序。现在依然是使用 《用C++实现类似于JAVA HttpServlet 的编程接口 》示例中的 http_servlet 类，只是稍微修改一下 main 函数，就变成下面的情形：

```c++
//////////////////////////////////////////////////////////////////////////

class master_service : public acl::master_proc
{
public:
	master_service() {}
	~master_service() {}
protected:
	// 基类虚函数，当接收到一个 HTTP 客户端请求时，服务器
	// 框架回调此函数将连接流传入
	virtual void on_accept(acl::socket_stream* stream)
	{
		// HttpServlet 的子类实例
		http_servlet servlet("127.0.0.1:11211");
		servlet.setLocalCharset("gb2312");  // 设置本地字符集
		servlet.doRun(stream);  // 开始处理浏览器请求过程
	}
};

//////////////////////////////////////////////////////////////////////////

int main(int argc, char* argv[])
{
	acl::acl_cpp_init();  // 初始化 acl_cpp 库
	master_service service;  // 半驻留进程池服务类对象

	// 开始运行

	if (argc >= 2 && strcmp(argv[1], "alone") == 0)
	{
		// 当在手工调试时一般采用此方式
		printf("listen: 127.0.0.1:8888 ...\r\n");
		service.run_alone("127.0.0.1:8888", NULL, 1);  // 单独运行方式
	}
	else  // 生产环境中以半驻留进程池模式运行
		service.run_daemon(argc, argv);  // acl_master 控制模式运行

	return 0;
}
```

上面的例子是一个结合 HttpServlet 类及 master_service 进程池服务类的 HTTP 服务器程序，关于进程池的例子，可以先结合本人原来写过的基于C语言库 acl 的一篇文章《快速创建你的服务器程序－－single进程池模型 》。

不仅可以非常容易地将 HttpServlet 写成进程池方式，同时还可以结合 acl_cpp 的线程池框架模板，将 HttpServlet 类实现为半驻留线程池实例，下面就显示了这一过程：

```c++
class master_threads_test : public acl::master_threads
{
public:
	master_threads_test() {}

	~master_threads_test() {}

protected:
	// 基类纯虚函数：当客户端连接有数据可读或关闭时回调此函数，返回 true 表示
	// 继续与客户端保持长连接，否则表示需要关闭客户端连接
	bool thread_on_read(acl::socket_stream* stream)
	{
		// HttpServlet 的子类实例
		http_servlet servlet;
		servlet.setLocalCharset("gb2312");  // 设置本地字符集
		servlet.doRun(“127.0.0.1：11211”， stream);  // 开始处理浏览器请求过程，同时设置 memcached 的监听地址及客户端连接流
	}

	// 基类虚函数：当接收到一个客户端请求时，调用此函数，允许
	// 子类事先对客户端连接进行处理，返回 true 表示继续，否则
	// 要求关闭该客户端连接
	bool thread_on_accept(acl::socket_stream*)
	{
		return true;  // 返回 true 以允许服务器框架继续调用 thread_on_read 过程
	}
};

// 字符串类配置参数项

static char *var_cfg_debug_msg;

static acl::master_str_tbl var_conf_str_tab[] = {
	{ "debug_msg", "test_msg", &var_cfg_debug_msg },

	{ 0, 0, 0 }
};

// 布尔配置参数项
static int  var_cfg_debug_enable;
static int  var_cfg_keep_alive;
static int  var_cfg_loop;

static acl::master_bool_tbl var_conf_bool_tab[] = {
	{ "debug_enable", 1, &var_cfg_debug_enable },
	{ "keep_alive", 1, &var_cfg_keep_alive },
	{ "loop_read", 1, &var_cfg_loop },

	{ 0, 0, 0 }
};

// 整数配置参数项
static int  var_cfg_io_timeout;

static acl::master_int_tbl var_conf_int_tab[] = {
	{ "io_timeout", 120, &var_cfg_io_timeout, 0, 0 },

	{ 0, 0 , 0 , 0, 0 }
};

int main(int argc, char* argv[])
{
	master_threads_test mt;  // 半驻留线程池服务器实例

	// 设置配置参数表
	mt.set_cfg_int(var_conf_int_tab);
	mt.set_cfg_int64(NULL);
	mt.set_cfg_str(var_conf_str_tab);
	mt.set_cfg_bool(var_conf_bool_tab);

	// 开始运行

	if (argc >= 2 && strcmp(argv[1], "alone") == 0)
	{
		// 当在手工调试时一般采用此方式
		printf("listen: 127.0.0.1:8888\r\n");
		mt.run_alone("127.0.0.1:8888", NULL, 2, 10);  // 单独运行方式
	}
	else  // 生产环境中以半驻留线程池模式运行
		mt.run_daemon(argc, argv);  // acl_master 控制模式运行
	return 0;
}
```
