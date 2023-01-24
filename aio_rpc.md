---
title: acl_cpp 的 rpc 相关类整合阻塞及非阻塞过程
date: 2012-07-13 23:03
categories: 非阻塞编程
---

## 一、概述

非阻塞网络编程无疑成了高并发、高性能编程的代名词，但现实应用编程中并不是每种应用都需要采用非阻塞编程模式，因为这将大大增加编程的复杂性、开发周期以及出错率，所以我们写的绝大部分网络程序程序都是阻塞的，一般是一个进程一个网络连接或一个线程一个网络连接。即然非阻塞模式可以实现高并发网络连接，阻塞模式可以实现复杂的业务逻辑，那是否有办法将二者结合起来呢？答案是肯定的，其中在 acl_cpp 库中，ipc 目录下的模块就是为了满足这种需求而设计的。

## 二、接口设计

在 rpc.hpp 头文件中有两个类：rpc_service 和 rpc_request，其中 rpc_request 是一个纯虚类，用户需要继承此类并实现该类中规定的纯虚接口，从而实现自己的阻塞操作功能；rpc_service 是阻塞与非阻塞结合的粘合类，通过将 rpc_request 子类实例传递给 rpc_service 类实例的 rpc_fork 方法，实现将阻塞请求过程交给子线程处理的目的。

### 1、rpc_service 类

#### 1.1、构造函数：
```c++
	/**
	 * 构造函数
	 * @param nthread {int} 如果该值 > 1 则内部自动采用线程池，否则
	 *  则是一个请求一个线程
	 * @param win32_gui {bool} 是否是窗口类的消息，如果是，则内部的
	 *  通讯模式自动设置为基于 WIN32 的消息，否则依然采用通用的套接
	 *  口通讯方式
	 */
	rpc_service(int nthread, bool win32_gui = false)
```

从参数 win32_gui 可以看出，acl_cpp 的阻塞/非阻塞结合模式不仅可以用在通常的网络编程中，同时还可以用在阻塞过程与 WINDOWS 界面消息相结合的方面，这无非是经常用 MFC 进行界面编程的福音（例如，用户 VC 写了一个界面程序---当然这个界面窗口是基于 WIN32 消息的，但如果想进行一些数据库操作或文件下载操作，对用户而言阻塞式方法是非常容易实现这两类需求的，但 WIN32 的界面过程不能堵在任何一个数据库操作或文件下载过程，原来 VC 程序员通常的做法也是创建单独的线程进行阻塞操作，然后通过给主窗口传递消息将操作结果通知至主线程，幸运的是 acl_cpp 的 rpc 相关类可以使这一过程更为方便快捷；再如，你写的一个网络服务器程序的主线程是非阻塞的，但其中你不得不调用别人提供的库以实现用户身份验证的功能，同时这个用于用户认证的库又恰恰是阻塞的---一般也是如此，固然，你也许可以费很周折实现这一过程，同样，acl_cpp 的 rpc 相关类可以帮你解决这类问题）。

#### 1.2、将阻塞处理过程的对象交由子线程处理：
```c++
	/**
	 * 主线程中运行：将请求任务放入子线程池的任务请求队列中，当线程池
	 * 中的一个子线程接收到该任务后便调用 rpc_request::rpc_run 方法调
	 * 用子类的方法，当任务处理完毕后给主线程发消息，在主线程中再调用
	 * rpc_request::rpc_callback
	 * @param req {rpc_request*} rpc_request 子类实例，非空
	 */
	void rpc_fork(rpc_request* req);
```

通过此接口，可以将阻塞请求过程交给子线程处理，子线程处理完后再通知主线程。

### 2、rpc_request 类

在类 rpc_request 中有三个虚接口，用户子类必须实现其中的两个纯虚接口：rpc_run 和 rpc_onover，同时用户可以根据需要实现另一非纯虚接口：rpc_wakeup。是当用户调完 rpc_service::rpc_fork 且子线程接收到此请求任务后 rpc_request::rpc_run 方法会被调用（一定切记：rpc_fork 是在主线程中被调用的，而 rpc_run 是在子线程中被调用的）；当 rpc_request::rpc_run 函数返回后，rpc_reuqest::rpc_onover 会在主线程中被调用以表示子线程已经处理完毕（同样需要严重注意：rpc_request::rpc_onover 方法又回到主线程中被调用），这样，通过这两个过程就实现了将阻塞过程放在子线程中处理，主线程的非阻塞过程（非阻塞网络事件或非阻塞的 WIN32 消息过程）异步地等待子线程完成任务。

```c++
	/**
	 * 在子线程中被调用，子类必须实现此接口，用于处理具体任务
	 */
	virtual void rpc_run(void) = 0;

	/**
	 * 在主线程中被调用，子类必须实现此接口，
	 * 当子线程处理完请求任务后该接口将被调用，所以该类对象只能
	 * 是当本接口调用后才能被释放，禁止在调用本接口前释放本类对象
	 */
	virtual void rpc_onover(void) = 0;
```

另外，在 rpc_request 类中还有一个非纯虚方法：rpc_wakeup，这是做什么用的呢？可以假设这种应用场景，在子线程中调用 rpc_request::rpc_run 方法内部过程中，用户如果需要通知主线程一些中间状态（比如，文件下载的进度）该怎么办？那就在 rpc_run 方法内先调用 rpc_request::rpc_signal 通知主线程子线程处理的中间状态，则在主线程中用户实现的 rpc_request::rpc_wakeup 虚接口就会被调用。下面是 rpc_signal 和 rpc_wakeup 的接口说明：

```c++
	/**
	 * 在子线程中被调用，内部自动支持套接口或 WIN32 窗口消息
	 * 子类实例的 rpc_run 方法中可以多次调用此方法向主线程的
	 * 本类实例发送消息，主线程中调用本对象 rpc_wakeup 方法
	 * @param ctx {void*} 传递的参数指针，一般应该是动态地址
	 *  比较好，这样可以避免同一个参数被重复覆盖的问题
	 */
	void rpc_signal(void* ctx);

	/**
	 * 虚接口：当子线程调用本对象的 rpc_signal 时，在主线程中会
	 * 调用本接口，通知在任务未完成前(即调用 rpc_onover 前)收到
	 * 子线程运行的中间状态信息；内部自动支持套接口或 WIN32 窗口
	 * 消息；应用场景，例如，对于 HTTP 下载应用，在子线程中可以
	 * 一边下载，一边向主线程发送(调用 rpc_signal 方法)下载进程，
	 * 则主线程会调用本类实例的此方法来处理此消息
	 */
	virtual void rpc_wakeup(void* ctx) { (void) ctx; }
```

紧接着这个应用场景，假设在子线程调用 rpc_run 的内部通过 rpc_signal 通知主线程的中间状态后，希望主线程能收到此通知消息并且希望得到主线程下一步希望执行的指令才会进一步继续执行。于是便有了 rpc_request::cond_wait 和 rpc_request::cond_signal 两个方法的产生，即子线程通过 cond_wait 阻塞地等待主线程的下一步操作指令，主线程则调用 cond_signal 通知子线程下步指令，下面是这两个方法的说明：

```c++
	/**
	 * 当子线程调用 rpc_signal 给主线程后，调用本方法可以等待
	 * 主线程发来下一步指令
	 * @param timeout {int} 等待超时时间(毫秒)，当该值为 0 时
	 *  则采用非阻塞等待模式，当该值为 < 0 时，则采用完全阻塞
	 *  等待模式(即一直等到主线程发送 cond_signal 通知)，当该
	 *  值 > 0 时，则等待的最大超时时间为 timeout 毫秒
	 * @return {bool} 返回 true 表示收到主线程发来的通知信号，
	 *  否则，需要调用 cond_wait_timeout 判断是否是超时引起的
	 */
	bool cond_wait(int timeout = -1);

	/**
	 * 在子线程中被调用，内部自动支持套接口或 WIN32 窗口消息
	 * 子类实例的 rpc_run 方法中可以多次调用此方法向主线程的
	 * 本类实例发送消息，主线程中调用本对象 rpc_wakeup 方法
	 * @param ctx {void*} 传递的参数指针，一般应该是动态地址
	 *  比较好，这样可以避免同一个参数被重复覆盖的问题
	 */
	void rpc_signal(void* ctx);
```

## 三、示例

如果您能大体明白上面有关 rpc_service 和 rpc_request 类的功能说明，相信下面的例子您也一定能看明白：

```c++
// rpc_download.cpp : 定义控制台应用程序的入口点。
//

#include "stdafx.h"
#include <assert.h>
#include "lib_acl.hpp"

using namespace acl;

typedef enum
{
	CTX_T_CONTENT_LENGTH,
	CTX_T_PARTIAL_LENGTH,
	CTX_T_END
} ctx_t;

struct DOWN_CTX 
{
	ctx_t type;
	int length;
};

static int __download_count = 0;

class http_download : public rpc_request
{
public:
	http_download(aio_handle& handle, const char* addr, const char* url)
		: handle_(handle)
		, addr_(addr)
		, url_(url)
		, error_(false)
		, total_read_(0)
		, content_length_(0)
	{}
	~http_download() {}
protected:

	// 子线程处理函数
	void rpc_run()
	{
		http_request req(addr_);  // HTTP 请求对象

		// 设置 HTTP 请求头信息
		req.request_header().set_url(url_.c_str())
			.set_content_type("text/html")
			.set_host(addr_.c_str())
			.set_method(HTTP_METHOD_GET);

		// 测试用，显示 HTTP 请求头信息内容
		string header;
		req.request_header().build_request(header);
		printf("request: %s\r\n", header.c_str());

		// 发送 HTTP 请求数据
		if (req.request(NULL, 0) == false)
		{
			printf("send request error\r\n");
			error_ = false;
			return;
		}

		// 获得 HTTP 请求的连接对象
		http_client* conn = req.get_client();
		assert(conn);
		DOWN_CTX* ctx = new DOWN_CTX;
		ctx->type = CTX_T_CONTENT_LENGTH;

		// 获得 HTTP 响应数据的数据体长度
		ctx->length = (int) conn->body_length();
		content_length_ = ctx->length;

		// 通知主线程
		rpc_signal(ctx);

		char buf[8192];
		while (true)
		{
			// 读 HTTP 响应数据体
			int ret = req.get_body(buf, sizeof(buf));
			if (ret <= 0)
			{
				ctx = new DOWN_CTX;
				ctx->type = CTX_T_END;
				ctx->length = ret;
				// 通知主线程
				rpc_signal(ctx);
				break;
			}
			ctx = new DOWN_CTX;
			ctx->type = CTX_T_PARTIAL_LENGTH;
			ctx->length = ret;
			// 通知主线程
			rpc_signal(ctx);
		}
	}

	// 主线程处理过程，收到子线程任务完成的消息
	void rpc_onover()
	{
		printf("%s: read over now, total read: %d, content-length: %d\r\n",
			addr_.c_str(), total_read_, content_length_);

		// 当 HTTP 响应都完成时，通知主线程停止事件循环过程
		__download_count--;
		if (__download_count == 0)
			handle_.stop();
	}

	// 主线程处理过程，收到子线程的通知消息
	void rpc_wakeup(void* ctx)
	{
		DOWN_CTX* down_ctx = (DOWN_CTX*) ctx;
		switch (down_ctx->type)
		{
		case CTX_T_CONTENT_LENGTH:
			printf("%s: content-length: %d\r\n",
				addr_.c_str(), down_ctx->length);
			break;
		case CTX_T_PARTIAL_LENGTH:
			total_read_ += down_ctx->length;
			printf("%s: partial-length: %d, total read: %d\r\n",
				addr_.c_str(), down_ctx->length, total_read_);
			break;
		case CTX_T_END:
			printf("%s: read over\r\n", addr_.c_str());
			break;
		default:
			printf("%s: ERROR\r\n", addr_.c_str());
			break;
		}
		delete down_ctx;
	}
private:
	aio_handle& handle_;
	string addr_;
	string url_;
	bool  error_;
	int   total_read_;
	int   content_length_;
};

static void run(void)
{
	aio_handle handle;
	rpc_service* service = new rpc_service(10);  // 创建 rpc 服务对象

	// 打开消息服务器
	if (service->open(&handle) == false)
	{
		printf("open service error: %s\r\n", last_serror());
		return;
	}

	// 下载页面内容

	http_download down1(handle, "www.sina.com.cn:80", "http://www.sina.com.cn/");
	service->rpc_fork(&down1);  // 发起一个阻塞会话过程
	__download_count++;

	http_download down2(handle, "www.hexun.com:80", "/");
	service->rpc_fork(&down2);  // 发起第二个阻塞会话过程
	__download_count++;

	// 异步事件循环过程
	while (true)
	{
		if (handle.check() == false)
			break;
	}

	delete service;
	handle.check(); // 保证释放所有延迟关闭的异步对象
}

int main(void)
{
#ifdef WIN32
	acl_cpp_init();
#endif

	run();
	printf("Enter any key to continue\r\n");
	getchar();
	return 0;
}
```

acl 下载:

github：https://github.com/acl-dev/acl
gitee：https://gitee.com/acl-dev/acl