---
title: 非阻塞网络编程实例讲解
date: 2012-04-04 23:22
categories: 非阻塞编程
---

## 一、概述    
acl 库的 C 库(lib_acl) 的 aio 模块设计了完整的非阻塞异步 IO 通信过程，在 acl 的C++库(lib_acl_cpp) 中封装并增强了异步通信的功能，本文主要描述了 acl C++ 库之非阻塞IO库的设计及使用方法，该异步流的设计思路为：异步流类与异步流接口类，其中异步流类对象完成网络套接口监听、连接、读写的操作，异步流接口类对象定义了网络读写成功/超时回调、连接成功回调、接收客户端连接回调等接口；用户在进行异步编程时，首先必须实现接口类中定义的纯方法，然后将接口类对象在异步流对象中进行注册，这样当满足接口类对象的回调条件时 acl_cpp 的异步框架便自动调用用户定义的接口方法。

在 acl_cpp 中异步流的类继续关系如下图所示：
![异步流类继承关系图](/img/aio_inherit.png)

由上图可以看出，基类 aio_stream 中定义了流关闭，注册/取消流关闭回调和流超时回调等基础方法；aio_istream 和 aio_ostream 分别定义了异步流读及写的基本方法，aio_istream 中包含添加/删除流读成功回调接口类对象的方法，aio_ostream 中包含添加/删除流写成功回调接口类对象的方法；aio_socket_stream 类对象为连接服务器成功后的客户端流，或服务器接收到客户端连接创建的客户端连接流，其中定义了做为连接流时远程连接的方法及添加连接成功回调接口的方法；aio_listen_stream 类为监听流类，其中定义了监听某个网络地址（或UNIX下的域套接口地址）方法，以及注册接收成功接口的方法。

acl_cpp 异步流接口类继承关系图如下图：
![异步流类继承关系图](/img/aio_callback.png)

异步流接口类的设计中：aio_accept_callback 为监听流的回调接口类，用户应继承该类以获得外来客户端连接流，同时还需要定义继承于 aio_callback 的类，用于获得网络读写操作等结果信息；aio_open_callback 只有当客户端连接远程服务器时，用户需要实现其子类获得连接成功的结果。

## 二、实例
### 1、异步服务器

```c++
#include <iostream>
#include <assert.h>
#include "aio_handle.hpp"
#include "aio_istream.hpp"
#include "aio_listen_stream.hpp"
#include "aio_socket_stream.hpp"

using namespace acl;

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
		std::cout << "delete io_callback now ..." << std::endl;
	}

	/**
	 * 实现父类中的虚函数，客户端流的读成功回调过程
	 * @param data {char*} 读到的数据地址
	 * @param len {int} 读到的数据长度
	 * @return {bool} 返回 true 表示继续，否则希望关闭该异步流
	 */
	bool read_callback(char* data, int len)
	{
		i_++;
		if (i_ < 10)
			std::cout << ">>gets(i:" << i_ << "): " << data;

		// 如果远程客户端希望退出，则关闭之
		if (strncasecmp(data, "quit", 4) == 0)
		{
			client_->format("Bye!\r\n");
			client_->close();
		}

		// 如果远程客户端希望服务端也关闭，则中止异步事件过程
		else if (strncasecmp(data, "stop", 4) == 0)
		{
			client_->format("Stop now!\r\n");
			client_->close();  // 关闭远程异步流

			// 通知异步引擎关闭循环过程
			client_->get_handle().stop();
		}

		// 向远程客户端回写收到的数据

		client_->write(data, len);

		return (true);
	}

	/**
	 * 实现父类中的虚函数，客户端流的写成功回调过程
	 * @return {bool} 返回 true 表示继续，否则希望关闭该异步流
	 */
	bool write_callback()
	{
		return (true);
	}

	/**
	 * 实现父类中的虚函数，客户端流的超时回调过程
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
		std::cout << "Timeout ..." << std::endl;
		return (true);
	}

private:
	aio_socket_stream* client_;
	int   i_;
};

/**
 * 异步监听流的回调类的子类
 */
class io_accept_callback : public aio_accept_callback
{
public:
	io_accept_callback() {}
	~io_accept_callback()
	{
		printf(">>io_accept_callback over!\n");
	}

	/**
	 * 基类虚函数，当有新连接到达后调用此回调过程
	 * @param client {aio_socket_stream*} 异步客户端流
	 * @return {bool} 返回 true 以通知监听流继续监听
	 */
	bool accept_callback(aio_socket_stream* client)
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

		// 从异步流读一行数据
		client->gets(10, false);
		return (true);
	}
};

int main(int argc, char* argv[])
{
	// 初始化ACL库(尤其是在WIN32下一定要调用此函数，在UNIX平台下可不调用)
	acl_cpp_init();

	// 构建异步引擎类对象
	aio_handle handle(ENGINE_KERNEL);

	// 创建监听异步流
	aio_listen_stream* sstream = new aio_listen_stream(&handle);
	const char* addr = "127.0.0.1:9001";

	// 监听指定的地址
	if (sstream->open(addr) == false)
	{
		std::cout << "open " << addr << " error!" << std::endl;
		sstream->close();
		// XXX: 为了保证能关闭监听流，应在此处再 check 一下
		handle.check();

		getchar();
		return (1);
	}

	// 创建回调类对象，当有新连接到达时自动调用此类对象的回调过程
	io_accept_callback callback;
	sstream->add_accept_callback(&callback);
	std::cout << "Listen: " << addr << " ok!" << std::endl;

	while (true)
	{
		// 如果返回 false 则表示不再继续，需要退出
		if (handle.check() == false)
		{
			std::cout << "aio_server stop now ..." << std::endl;
			break;
		}
	}

	// 关闭监听流并释放流对象
	sstream->close();

	// XXX: 为了保证能关闭监听流，应在此处再 check 一下
	handle.check();

	return (0);
}
```

简要说明一下，上面代码的基本思路是：

- 创建异步通信框架对象 aio_handle --> 创建异步监听流 aio_listen_stream 并注册回调类对象 io_accept_callback-->进入异步通信框架的事件循环中；
- 当接收到客户端连接后，异步框架回调 io_accept_callback 类对象的 accept_callback 接口并将客户端异步流输入-->创建异步流接口类对象，并将该对象注册至客户端异步流对象中;
- 当客户端异步流收到数据时回调异步流接口中的 read_callback 方法 --> 回写收到数据至客户端；当客户端流连接关闭时回调异步流接口中的close_callback --> 如果该接口类对象是动态创建的则需要手工 delete 掉；当接收客户端数据超时时会回调异步流接口中的 time_callback，该函数如果返回 true 则表示希望异步框架不关闭该客户端异步流，否则则关闭。

异步监听流的接口类的纯虚函数：virtual bool accept_callback(aio_socket_stream* client)  需要子类实现，子类在该函数中获得客户端连接异步流对象。

客户端异步流接口类 aio_callback 有四个虚函数：
- `virtual bool read_callback(char* data, int len)`  当客户端异步流读到数据时的回调虚函数；
- `virtual bool write_callback()` 当客户端异步流写数据成功后的回调虚函数；
- `virtual void close_callback()` 当异步流(客户端流或监听流)关闭时的回调虚函数；
- `virtual bool timeout_callback()` 当异步流（客户端流在读写超时或监听流在监听超时）超时时的回调函数虚函数。

### 2、异步客户端
```c++
#include <iostream>
#include <assert.h>
#include "string.hpp"
#include "util.hpp"
#include "aio_handle.hpp"
#include "acl_cpp_init.hpp"
#include "aio_socket_stream.hpp"

#ifdef WIN32
# ifndef snprintf
#  define snprintf _snprintf
# endif
#endif

using namespace acl;

typedef struct
{
	char  addr[64];
	aio_handle* handle;
	int   connect_timeout;
	int   read_timeout;
	int   nopen_limit;
	int   nopen_total;
	int   nwrite_limit;
	int   nwrite_total;
	int   nread_total;
	int   id_begin;
	bool  debug;
} IO_CTX;

static bool connect_server(IO_CTX* ctx, int id);

/**
 * 客户端异步连接流回调函数类
 */
class client_io_callback : public aio_open_callback
{
public:
	/**
	 * 构造函数
	 * @param ctx {IO_CTX*}
	 * @param client {aio_socket_stream*} 异步连接流
	 * @param id {int} 本流的ID号
	 */
	client_io_callback(IO_CTX* ctx, aio_socket_stream* client, int id)
		: client_(client)
		, ctx_(ctx)
		, nwrite_(0)
		, id_(id)
	{
	}

	~client_io_callback()
	{
		std::cout << ">>>ID: " << id_ << ", io_callback deleted now!" << std::endl;
	}

	/**
	 * 基类虚函数, 当异步流读到所要求的数据时调用此回调函数
	 * @param data {char*} 读到的数据地址
	 * @param len {int｝ 读到的数据长度
	 * @return {bool} 返回给调用者 true 表示继续，否则表示需要关闭异步流
	 */
	bool read_callback(char* data, int len)
	{
		(void) data;
		(void) len;

		ctx_->nread_total++;

		if (ctx_->debug)
		{
			if (nwrite_ < 10)
				std::cout << "gets(" << nwrite_ << "): " << data;
			else if (nwrite_ % 2000 == 0)
				std::cout << ">>ID: " << id_ << ", I: "
					<< nwrite_ << "; "<<  data;
		}

		// 如果收到服务器的退出消息，则也应退出
		if (acl::strncasecmp_(data, "quit", 4) == 0)
		{
			// 向服务器发送数据
			client_->format("Bye!\r\n");
			// 关闭异步流连接
			client_->close();
			return (true);
		}

		if (nwrite_ >= ctx_->nwrite_limit)
		{
			if (ctx_->debug)
				std::cout << "ID: " << id_
					<< ", nwrite: " << nwrite_
					<< ", nwrite_limit: " << ctx_->nwrite_limit
					<< ", quiting ..." << std::endl;

			// 向服务器发送退出消息
			client_->format("quit\r\n");
			client_->close();
		}
		else
		{
			char  buf[256];
			snprintf(buf, sizeof(buf), "hello world: %d\n", nwrite_);
			client_->write(buf, (int) strlen(buf));

			// 向服务器发送数据
			//client_->format("hello world: %d\n", nwrite_);
		}

		return (true);
	}

	/**
	 * 基类虚函数, 当异步流写成功时调用此回调函数
	 * @return {bool} 返回给调用者 true 表示继续，否则表示需要关闭异步流
	 */
	bool write_callback()
	{
		ctx_->nwrite_total++;
		nwrite_++;

		// 从服务器读一行数据
		client_->gets(ctx_->read_timeout, false);
		return (true);
	}

	/**
	 * 基类虚函数, 当该异步流关闭时调用此回调函数
	 */
	void close_callback()
	{
		if (client_->is_opened() == false)
		{
			std::cout << "Id: " << id_ << " connect "
				<< ctx_->addr << " error: "
				<< acl::last_serror();

			// 如果是第一次连接就失败，则退出
			if (ctx_->nopen_total == 0)
			{
				std::cout << ", first connect error, quit";
				/* 获得异步引擎句柄，并设置为退出状态 */
				client_->get_handle().stop();
			}
			std::cout << std::endl;
			delete this;
			return;
		}

		/* 获得异步引擎中受监控的异步流个数 */
		int nleft = client_->get_handle().length();
		if (ctx_->nopen_total == ctx_->nopen_limit && nleft == 1)
		{
			std::cout << "Id: " << id_ << " stop now! nstream: "
				<< nleft << std::endl;
			/* 获得异步引擎句柄，并设置为退出状态 */
			client_->get_handle().stop();
		}

		// 必须在此处删除该动态分配的回调类对象以防止内存泄露
		delete this;
	}

	/**
	 * 基类虚函数，当异步流超时时调用此函数
	 * @return {bool} 返回给调用者 true 表示继续，否则表示需要关闭异步流
	 */
	bool timeout_callback()
	{
		std::cout << "Connect " << ctx_->addr << " Timeout ..." << std::endl;
		client_->close();
		return (false);
	}

	/**
	 * 基类虚函数, 当异步连接成功后调用此函数
	 * @return {bool} 返回给调用者 true 表示继续，否则表示需要关闭异步流
	 */
	bool open_callback()
	{
		// 连接成功，设置IO读写回调函数
		client_->add_read_callback(this);
		client_->add_write_callback(this);
		ctx_->nopen_total++;

		acl::assert_(id_ > 0);
		if (ctx_->nopen_total < ctx_->nopen_limit)
		{
			// 开始进行下一个连接过程
			if (connect_server(ctx_, id_ + 1) == false)
				std::cout << "connect error!" << std::endl;
		}

		// 异步向服务器发送数据
		//client_->format("hello world: %d\n", nwrite_);
		char  buf[256];
		snprintf(buf, sizeof(buf), "hello world: %d\n", nwrite_);
		client_->write(buf, (int) strlen(buf));

		// 异步从服务器读取一行数据
		client_->gets(ctx_->read_timeout, false);

		// 表示继续异步过程
		return (true);
	}

private:
	aio_socket_stream* client_;
	IO_CTX* ctx_;
	int   nwrite_;
	int   id_;
};

static bool connect_server(IO_CTX* ctx, int id)
{
	// 开始异步连接远程服务器
	aio_socket_stream* stream = aio_socket_stream::open(ctx->handle,
			ctx->addr, ctx->connect_timeout);
	if (stream == NULL)
	{
		std::cout << "connect " << ctx->addr << " error!" << std::endl;
		std::cout << "stoping ..." << std::endl;
		if (id == 0)
			ctx->handle->stop();
		return (false);
	}

	// 创建连接后的回调函数类
	client_io_callback* callback = new client_io_callback(ctx, stream, id);

	// 添加连接成功的回调函数类
	stream->add_open_callback(callback);

	// 添加连接失败后回调函数类
	stream->add_close_callback(callback);

	// 添加连接超时的回调函数类
	stream->add_timeout_callback(callback);
	return (true);
}

int main(int argc, char* argv[])
{
	bool use_kernel = false;
	int   ch;
	IO_CTX ctx;

	memset(&ctx, 0, sizeof(ctx));
	ctx.connect_timeout = 5;
	ctx.nopen_limit = 10;
	ctx.id_begin = 1;
	ctx.nwrite_limit = 10;
	ctx.debug = false;
	snprintf(ctx.addr, sizeof(ctx.addr), "127.0.0.1:9001");

	acl_cpp_init();

	aio_handle handle(ENGINE_KERNEL);
	ctx.handle = &handle;

	if (connect_server(&ctx, ctx.id_begin) == false)
	{
		std::cout << "enter any key to exit." << std::endl;
		getchar();
		return (1);
	}

	std::cout << "Connect " << ctx.addr << " ..." << std::endl;

	while (true)
	{
		// 如果返回 false 则表示不再继续，需要退出
		if (handle.check() == false)
			break;
	}

	return (0);
}
```

异步客户端的基本流程为：
- 创建异步框架对象 aio_handle --> 异步连接远程服务器，创建连接成功/失败/超时的异步接口类对象并注册至异步连接流中 --> 异步框架进行事件循环中；
- 连接成功后，异步接口类对象中的 open_callback 被调用，启动下一个异步连接过程（未达限制连接数前） --> 添加异步读及异步写的回调接口类  --> 异步写入数据，同时开始异步读数据过程；
- 当客户端异步流收到数据时回调异步流接口中的 read_callback 方法 --> 回写收到数据至客户端；当客户端流连接关闭时回调异步流接口中的 close_callback --> 如果该接口类对象是动态创建的则需要手工 delete 掉；当接收客户端数据超时时会回调异步流接口中的 time_callback，该函数如果返回 true 则表示希望异步框架不关闭该客户端异步流，否则则关闭。

客户端异步连接流的接口类 aio_open_callback 的纯虚函数 virtual bool open_callback() 需要子类实现，在连接服务器成功后调用此函数，允许子类在该函数中做进一步操作，如：注册客户端流的异步读回调接口类对象及异步写回调类对象；如果连接超时或连接失败而导致的关闭，则基础接口类中的 timeout_callback() 或 close_callback() 将会被调用，以通知用户应用程序。

## 三、小结
以上的示例演示了基本的非阻塞异步流的监听、读写、连接等过程，类的设计中也提供了基本的操作方法，为了应对实践中的多样性及复杂性，acl_cpp 的异步流还设计了更多的接口和方法，如：延迟读写操作（这对于限流的服务器比较有用处）、定时器操作等。

更多例子参见：lib_acl_cpp/samples/aio/ 目录
github：https://github.com/acl-dev/acl
gitee：https://gitee.com/acl-dev/acl