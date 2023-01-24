---
title: 使用 acl 库 rpc 功能类实现 阻塞任务过程与MFC 界面过程分离
date: 2013-02-24 16:42
categories: 非阻塞编程
---

## 一、概述

MFC 程序员在编写 Windows 界面程序时经常需要处理一些阻塞任务过程，为了避免阻塞窗口的消息过程，一般会将阻塞过程将由一个子线程处理，该子线程在处理过程中通过向界面线程发送 Windows 窗口消息将处理结果传递给窗口线程。在 acl 库中的 rpc 功能类实现了更为方便的处理方式，通过 rpc 功能类，用户可以在主线程中进行非阻塞过程（如：界面消息过程或网络非阻塞通讯过程），而将阻塞任务交由子线程处理（如：网络阻塞通讯或数据库操作等），子线程可以将任务处理的中间状态和最终状态通过 rpc 功能类传递给主线程。

acl 的 rpc 类不仅能实现网络通讯方面的阻塞与非阻塞的粘合，同时还实现了阻塞过程与 MFC 界面过程的粘合，本文将以一个具体的 HTTP 下载过程为例来描述这一过程（示例在 acl 库中的 acl/lib_acl_cpp/samples/gui_rpc 目录下）。关于 acl 库中 rpc 相关类的使用，用户可以参考 《acl_cpp 的 rpc 相关类整合阻塞及非阻塞过程》（在该文中的例子描述了非阻塞主线程与阻塞子线程的交互过程，其示例代码适用于 win32 及 linux 平台）。

## 二、实例

### 1、在界面主线程中初始化时创建 rpc 服务对象：acl::rpc_service

```c++
	// 全局静态变量
	static acl::aio_handle* handle_;
	static acl::rpc_service* service_;

	......

	// 创建非阻塞框架句柄，并采用 WIN32 消息模式：acl::ENGINE_WINMSG
	handle_ = new acl::aio_handle(acl::ENGINE_WINMSG);

	// 创建 rpc 服务对象
	int max_threads = 10;  // 服务最大子线程数量
	service_ = new acl::rpc_service(max_threads, true);
	// 打开消息服务
	if (service_->open(handle_) == false)
		logger_fatal("open service error: %s", acl::last_serror());
```

在上面代码中，有几点需要注意：1）创建的 rpc 服务对象是全局性的；2）在创建非阻塞句柄时必须指定为 win32 界面消息事件类型：acl::ENGINE_WINMSG；3）在创建 rpc_service 时的第二个参数 win32_gui 必须为 true。

### 2、创建 rpc 中 acl::rpc_request 类的子类，以实现阻塞非阻塞粘合过程
本例中该子类为：http_download，在 http_download 类中必须实现父类 acl::rpc_request 中定义的两个纯虚接口：rpc_run，rpc_onover。

其中，http_download 的头文件如下：

```c++
/**
 * http 请求过程类，该类对象在子线程中发起远程 HTTP 请求过程，将处理结果
 * 返回给主线程
 */
class http_download : public acl::rpc_request
{
public:
	/**
	 * 构造函数
	 * @param addr {const char*} HTTP 服务器地址，格式：domain:port
	 * @param url {const char*} http url 地址
	 * @param callback {rpc_callback*} http 请求结果通过此类对象
	 *  通知主线程过程
	 */
	http_download(const char* addr, const char* url,
		rpc_callback* callback);
protected:
	~http_download() {}

	// 基类虚函数：子线程处理函数
	virtual void rpc_run();

	// 基类虚函数：主线程处理过程，收到子线程任务完成的消息
	virtual void rpc_onover();

	// 基类虚函数：主线程处理过程，收到子线程的通知消息
	virtual void rpc_wakeup(void* ctx);

        ......
```

在 http_download 类的构造参数中有一个接口类：rpc_callback，这是一个纯虚类，主要是为了方便将 http 的结果数据返回给主线程，该类的声明如下：

```c++
// 纯虚类，子类须实现该类中的纯虚接口
class rpc_callback
{
public:
	rpc_callback() {}
	virtual ~rpc_callback() {}

	// 设置 HTTP 请求头数据虚函数
	virtual void SetRequestHdr(const char* hdr) = 0;
	// 设置 HTTP 响应头数据虚函数
	virtual void SetResponseHdr(const char* hdr) = 0;
	// 下载过程中的回调函数虚函数
	virtual void OnDownloading(long long int content_length,
		long long int total_read) = 0;
	// 下载完成时的回调函数虚函数
	virtual void OnDownloadOver(long long int total_read,
		double spent) = 0;
};
```

http_download 类的函数实现如下：

```c++
#include "stdafx.h"
#include <assert.h>
#include "http_download.h"

// 由子线程动态创建的 DOWN_CTX 对象的数据类型
typedef enum
{
	CTX_T_REQ_HDR,		// 为 HTTP 请求头数据
	CTX_T_RES_HDR,		// 为 HTTP 响应头数据
	CTX_T_CONTENT_LENGTH,	// 为 HTTP 响应体的长度
	CTX_T_PARTIAL_LENGTH,	// 为 HTTP 下载数据体的长度
	CTX_T_END
} ctx_t;

// 子线程动态创建的数据对象，主线程接收此数据
struct DOWN_CTX 
{
	ctx_t type;
	long long int length;
};

// 用来精确计算时间截间隔的函数，精确到毫秒级别
static double stamp_sub(const struct timeval *from,
	const struct timeval *sub_by)
{
	struct timeval res;

	memcpy(&res, from, sizeof(struct timeval));

	res.tv_usec -= sub_by->tv_usec;
	if (res.tv_usec < 0)
	{
		--res.tv_sec;
		res.tv_usec += 1000000;
	}

	res.tv_sec -= sub_by->tv_sec;
	return (res.tv_sec * 1000.0 + res.tv_usec/1000.0);
}

//////////////////////////////////////////////////////////////////////////

// 子线程处理函数
void http_download::rpc_run()
{
	acl::http_request req(addr_);  // HTTP 请求对象
	// 设置 HTTP 请求头信息
	req.request_header().set_url(url_.c_str())
		.set_content_type("text/html")
		.set_host(addr_.c_str())
		.set_method(acl::HTTP_METHOD_GET);

	req.request_header().build_request(req_hdr_);
	DOWN_CTX* ctx = new DOWN_CTX;
	ctx->type = CTX_T_REQ_HDR;
	rpc_signal(ctx);  // 通知主线程 HTTP 请求头数据

	struct timeval begin, end;;
	gettimeofday(&begin, NULL);

	// 发送 HTTP 请求数据
	if (req.request(NULL, 0) == false)
	{
		logger_error("send request error");
		error_ = false;
		gettimeofday(&end, NULL);
		total_spent_ = stamp_sub(&end, &begin);
		return;
	}

	// 获得 HTTP 请求的连接对象
	acl::http_client* conn = req.get_client();
	assert(conn);

	(void) conn->get_respond_head(&res_hdr_);
	ctx = new DOWN_CTX;
	ctx->type = CTX_T_RES_HDR;
	rpc_signal(ctx);   // 通知主线程 HTTP 响应头数据

	ctx = new DOWN_CTX;
	ctx->type = CTX_T_CONTENT_LENGTH;
	
	ctx->length = conn->body_length();  // 获得 HTTP 响应数据的数据体长度
	content_length_ = ctx->length;
	rpc_signal(ctx);  // 通知主线程 HTTP 响应体数据长度

	acl::string buf(8192);
	int   real_size;
	while (true)
	{
		// 读 HTTP 响应数据体
		int ret = req.read_body(buf, true, &real_size);
		if (ret <= 0)
		{
			ctx = new DOWN_CTX;
			ctx->type = CTX_T_END;
			ctx->length = ret;
			rpc_signal(ctx);  // 通知主线程下载完毕
			break;
		}
		ctx = new DOWN_CTX;
		ctx->type = CTX_T_PARTIAL_LENGTH;
		ctx->length = real_size;
		// 通知主线程当前已经下载的大小
		rpc_signal(ctx);
	}

	// 计算下载过程总时长
	gettimeofday(&end, NULL);
	total_spent_ = stamp_sub(&end, &begin);

	// 至此，子线程运行完毕，主线程的 rpc_onover 过程将被调用
}

//////////////////////////////////////////////////////////////////////////

http_download::http_download(const char* addr, const char* url,
	rpc_callback* callback)
	: addr_(addr)
	, url_(url)
	, callback_(callback)
	, error_(false)
	, total_read_(0)
	, content_length_(0)
	, total_spent_(0)
{

}

//////////////////////////////////////////////////////////////////////////

// 主线程处理过程，收到子线程任务完成的消息
void http_download::rpc_onover()
{
	logger("http download(%s) over, 共 %I64d 字节，耗时 %.3f 毫秒",
		url_.c_str(), total_read_, total_spent_);
	callback_->OnDownloadOver(total_read_, total_spent_);
	delete this;  // 销毁本对象
}

// 主线程处理过程，收到子线程的通知消息
void http_download::rpc_wakeup(void* ctx)
{
	DOWN_CTX* down_ctx = (DOWN_CTX*) ctx;

	// 根据子线程中传来的不同的下载阶段进行处理

	switch (down_ctx->type)
	{
	case CTX_T_REQ_HDR:
		callback_->SetRequestHdr(req_hdr_.c_str());
		break;
	case CTX_T_RES_HDR:
		callback_->SetResponseHdr(res_hdr_.c_str());
		break;
	case CTX_T_CONTENT_LENGTH:
		break;
	case CTX_T_PARTIAL_LENGTH:
		total_read_ += down_ctx->length;
		callback_->OnDownloading(content_length_, total_read_);
		break;
	case CTX_T_END:
		logger("%s: read over", addr_.c_str());
		break;
	default:
		logger_error("%s: ERROR", addr_.c_str());
		break;
	}

	// 删除在子线程中动态分配的对象
	delete down_ctx;
}

//////////////////////////////////////////////////////////////////////////
```
  

### 3、在 MFC 界面类中创建 rpc_callback 的子类，接收子线程的 HTTP 处理结果
本例直接将对话框类继承了 rpc_callback 接口类，其中部分内容如下：

```c++
// Cgui_rpcDlg 对话框
class Cgui_rpcDlg : public CDialog
	, public rpc_callback
{
// 构造
public:
	Cgui_rpcDlg(CWnd* pParent = NULL);	// 标准构造函数
	~Cgui_rpcDlg();

        ......

public:
	// 基类 rpc_callback 虚函数

	// 设置 HTTP 请求头数据虚函数
	virtual void SetRequestHdr(const char* hdr);
	// 设置 HTTP 响应头数据虚函数
	virtual void SetResponseHdr(const char* hdr);
	// 下载过程中的回调函数虚函数
	virtual void OnDownloading(long long int content_length,
		long long int total_read);
	// 下载完成时的回调函数虚函数
	virtual void OnDownloadOver(long long int total_read,
		double spent);
        ......
};
```

4、用 VC2003 编译该例子，运行可执行程序可以得到如下的界面：
![运行界面](/img/aio_rpc_mfc.png)

运行这个例子，在 URL 中输入地址（如：http://www.sina.com.cn ），点“开始运行”按钮，在下载 URL 数据的过程中移动界面窗口，可以看到界面窗口的消息过程并未被阻塞（因为 HTTP 阻塞下载过程是在子线程中进行的），同时界面的状态栏还能实时显示当前 URL 下载的进度状态（子线程通过 rpc_request 的消息传递方式将下载状态通知界面主线程）。

## 四、小结

在界面编程中，将阻塞过程与界面过程分离（ 即将阻塞过程交由子线程处理）是一种编程思想，不仅可以用在 PC 机的界面编程中，同时对于手机 APP 开发也有用处，这样做的好处是：一方面可以利用多核，更重要的是使得界面编程更为简单（要比所有模块全部采用非阻塞编程要容易得多）。

## 五、参考

github：https://github.com/acl-dev/acl
gitee：https://gitee.com/acl-dev/acl

 