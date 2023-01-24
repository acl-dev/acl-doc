---
title: 使用 acl 库开发简单的客户端与服务器程序
date: 2014-05-18 23:25
categories: 网络编程
---

acl 的 C++ 库部分也提供了一些简单的服务器类，本文将介绍如何使用这些简单的类来实现一些服务器程序和网络客户端程序。

首先介绍 acl 中的服务器类：server_socket。该类定义了如下几个简单方法：
 
```c++
	/**
	 * 开始监听给定服务端地址
	 * @param addr {const char*} 服务器监听地址，格式为：
	 *  ip:port；在 unix 环境下，还可以是域套接口，格式为：
	 *   /path/xxx
	 * @return {bool} 监听是否成功
	 */
	bool open(const char* addr);

	/**
	 * 关闭已经打开的监听套接口
	 * @return {bool} 是否正常关闭
	 */
	bool close();

	/**
	 * 接收客户端连接并创建客户端连接流
	 * @param timeout {int} 在阻塞模式下，当该值 > 0 时，采用超时
	 *  方式接收客户端连接，若在指定时间内未获得客户端连接，则返回 NULL
	 * @return {socket_stream*} 返回空表示接收失败
	 */
	socket_stream* accept(int timeout = 0);

	/**
	 * 获得监听的地址
	 * @return {const char*} 返回值非空指针
	 */
	const char* get_addr() const
	{
		return addr_;
	}
```

使用上述网络服务类的步骤是：调用 open 监听本机的一个网络地址（如果是UNIX平台，还可以监听UNIX域套接口）------> 调用 accept 方法等待远程客户端连接本服务器 ------> 当服务器程序接收到客户端连接时 accept 方法返回客户端连接网络流(socket_stream) ------> 启动一个线程处理这个客户端连接。下面为一个简单的服务器程序：

 
```c++
#include "acl_cpp/lib_acl.hpp"

// 处理客户端连接的线程类
class client_thread : public acl::thread
{
public:
	client_thread(acl::socket_stream* client)
	: client_(client)
	{
	}

	~client_thread()
	{
		delete client_;
	}

protected:
	// 实现基类 acl::thread 中定义的纯虚方法
	void* run()
	{
		acl::string buf;
		while (true)
		{
			// 从客户端连接读一行数据，第二个参数为 false 意思是希望
			// socket_stream 在读到一行数据时保留 \r\n
			if (client_->gets(buf, false) == false)
				return NULL;

			printf("gets one line: %s", buf.c_str());

			// 回写所读到的一行数据
			if (client_->write(buf) == -1)
				return NULL;
		}
	}

private:
	acl::socket_stream* client_;
};

int main(void)
{
	const char* addr = "0.0.0.0:8080";
	acl::socket_server server;

	// 监听本机网络地址
	if (server.open(addr) == false)
	{
		printf("listen addr: %s error: %s\r\n", addr, acl::last_serror());
		return -1
	}

	while (true)
	{
		// 等待客户端连接本服务器程序
		acl::socket_stream* client = server.accept();
		if (client == NULL)
		{
			printf("accept error: %s\r\n", acl::last_serror());
			return -1;
		}

		// 创建一个子线程用来处理该客户端连接
		client_thread* thread = new client_thread(client);

		// 将线程设为分离模式，这样当线程退出时会自行释放线程相关资源
		thread->set_detachable(true);

		// 启动该线程
		thread->start();
	}

	return 0;
}
```

上面例子非常简单，毋庸详述，关于如何使用 acl 编写多线程程序，请参照：使用 acl_cpp 库编写多线程程序。下面再给出一个简单的网络客户端例子：

```c++
#include "acl_cpp/lib_acl.hpp"

int main(void)
{
	const char* server_addr = "127.0.0.1:8080";
	int   conn_timeout = 10 /* 连接服务器超时时间，单位：秒 */
	int   rw_timeout = 10 /* 网络 IO 超时时间，单位：秒 */;
	acl::socket_stream conn;

	// 连接远程服务器
	if (conn.open(server_addr, conn_timeout, rw_timeout) == false)
	{
		printf("connect server: %s error: %s\r\n",
			server_addr, acl::last_serror());
		return -1;
	}

	const char* req = "hello world!\r\n";

	acl::string buf;

	// 向服务器写一行数据，同时从服务器读一行数据，循环 10 次
	for (int i = 0; i < 10; i++)
	{
		// 向服务器发送一行数据
		if (conn.write(req, strlen(req)) == -1)
		{
			printf("write request to server error: %s\r\n",
				acl::last_serror());
			return -1;
		}

		// 从服务器读一行数据，注：第二个参数为默认的 true，意思是获得
		// 一行数据后自动将尾部的 \r\n 去掉
		if (conn.gets(buf) == false)
		{
			printf("gets one line from server error: %s\r\n",
				acl::last_serror();
			return -1;
		}

		printf("response: %s\r\n", buf.c_str());
	}

	return 0;
}
``` 
