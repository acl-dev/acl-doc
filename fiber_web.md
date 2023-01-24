---
title: 使用协程方式编写高并发的WEB服务
date: 2016-07-06 21:01:24
tags: 协程编程
categories: 协程编程
---

在《使用 acl 协程编写高并发网络服务》中介绍了一个使用 acl 协程库编写高并发网络服务的应用示例，本节将展示一个稍微复杂些且更具实际意义的例子：基于协程的 WEB 服务器程序。下面首先展示这个 WEB 服务器程序：

```c++
#include "lib_acl.h"			// acl 基础库头文件
#include "fiber/lib_fiber.h"	// acl 协程库头文件
#include "acl_cpp/lib_acl.hpp"	// acl C++ 封装库头文件

class http_servlet : public acl::HttpServlet
{
public:
	http_servlet(acl::socket_stream* stream, acl::session* session)
			: HttpServlet(stream, session)
	{
	}

	~http_servlet(void)
	{
	}

	// override
	bool doGet(acl::HttpServletRequest& req, acl::HttpServletResponse& res)
	{
		return doPost(req, res);
	}

	// override
	bool doPost(acl::HttpServletRequest&, acl::HttpServletResponse& res)
	{
		const char* buf = "hello world!";
		size_t len = strlen(buf);

		res.setContentLength(len);
		res.setKeepAlive(true);

		// 发送 http 响应体
		return res.write(buf, len) && res.write(NULL, 0);
	}
};

#define	 STACK_SIZE	320000          // 指定协程堆栈大小(字节)
static int __rw_timeout = 0;		// 网络 IO 超时时间(秒)

static void http_server(ACL_FIBER *, void *ctx)
{
	acl::socket_stream *conn = (acl::socket_stream *) ctx;

	printf("start one http_server\r\n");

	acl::memcache_session session("127.0.0.1:11211");

	// 基于 ACL HTTP 模块的 Http 服务类
	http_servlet servlet(conn, &session);
	servlet.setLocalCharset("gb2312");

	// 循环处理客户端的 HTTP 请求
	while (true)
	{
		// 调用 acl::HttpServlet 类中的方法，从而触发子类重载的 doPost/doGet 方法
		if (servlet.doRun() == false)
			break;
	}

	printf("close one connection: %d, %s\r\n", conn->sock_handle(), acl::last_serror());

	// 销毁客户端连接对象
	delete conn;
}

static void fiber_accept(ACL_FIBER *, void *ctx)
{
	const char* addr = (const char* ) ctx;
	acl::server_socket server;

	// 监听本机服务端口
	if (server.open(addr) == false)
	{
		printf("open %s error\r\n", addr);
		exit (1);
	}
	else
		printf("open %s ok\r\n", addr);

	while (true)
	{
		// 等待接收外来 HTTP 客户端连接
		acl::socket_stream* client = server.accept();
		if (client == NULL)
		{
			printf("accept failed: %s\r\n", acl::last_serror());
			break;
		}

		client->set_rw_timeout(__rw_timeout);
		printf("accept one: %d\r\n", client->sock_handle());

		// 创建协程处理 HTTP 客户端连接请求
		acl_fiber_create(http_server, client, STACK_SIZE);
	}

	exit (0);
}

static void usage(const char* procname)
{
	printf("usage: %s -h [help] -s listen_addr -r rw_timeout\r\n", procname);
}

int main(int argc, char *argv[])
{
	acl::string addr("127.0.0.1:9001");
	int  ch;

	while ((ch = getopt(argc, argv, "hs:r:")) > 0)
	{
		switch (ch)
		{
		case 'h':
			usage(argv[0]);
			return 0;
		case 's':
			addr = optarg;
			break;
		case 'r':
			__rw_timeout = atoi(optarg);
			break;
		default:
			break;
		}
	}

	acl::acl_cpp_init();
	acl::log::stdout_open(true);

	// 创建服务监听协程
	acl_fiber_create(fiber_accept, addr.c_str(), STACK_SIZE);

	// 启动协程调度过程
	acl_fiber_schedule();

	return 0;
}
```

此例子的流程为：创建服务监听协程 ---> 启动协程调度过程 ---> 监听协程收到 HTTP 客户端连接后，创建 HTTP 协程处理该连接 ---> HTTP 协程与 HTTP 客户端进行交互。

因为协程相对于线程来说是非常轻量级的，所以虽然针对每一个 HTTP 客户端连接都会创建一个新的协程，但这并不会费太多系统资源，因而可以支持非常高的并发连接。

github：https://github.com/acl-dev/acl
gitee：http://git.oschina.net/acl-dev/acl
 