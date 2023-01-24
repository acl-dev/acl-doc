---
title: 使用 acl_cpp 库中的 http_request 类实现一个 HTTP 客户端请求的例子
date: 2014-06-02 20:06:24
tags: http
categories: http开发
---

之前写过几篇如何使用 acl 库来实现 HTTP 客户端的例子都是基于 C 语言(使用 acl 较为底层的 HTTP 协议库写 HTTP 下载客户端举例, 使用 acl 库开发一个 HTTP 下载客户端)，其实在 acl 的 C++ 库(lib_acl_cpp) 中 HTTP 类功能更为强大，本节将介绍如何使用 acl::http_request 类来写一些简单的 HTTP 客户端示例。

##  一、 acl::http_request 类的一些常用接口
该 HTTP 请求类有两个构造函数，如下 ：

```c++
	/**
	 * 构造函数：通过该构造函数传入的 socket_stream 流对象并
	 * 不会被关闭，需要调用者自己关闭
	 * @param client {socket_stream*} 数据连接流，非空，
	 *  在本类对象被销毁时该流对象并不会被销毁，所以用户需自行释放
	 * @param conn_timeout {int} 如果传入的流关闭，则内部会
	 *  自动重试，此时需要该值表示连接服务器的超时时间(秒)，
	 *  至于重连流的 IO 读写超时时间是从 输入的流中继承的
	 * @param unzip {bool} 是否对服务器响应的数据自动进行解压
	 * 注：当该类实例被多次使用时，用户应该在每次调用前调用
	 * request_header::http_header::reset()
	 */
	http_request(socket_stream* client, int conn_timeout = 60,
		bool unzip = true);

	/**
	 * 构造函数：该构造函数内部创建的 socket_stream 流会自行关闭
	 * @param addr {const char*} WEB 服务器地址
	 * @param conn_timeout {int} 远程连接服务器超时时间(秒)
	 * @param rw_timeout {int} IO 读写超时时间(秒)
	 * @param unzip {bool} 是否对服务器响应的数据自动进行解压
	 */
	http_request(const char* addr, int conn_timeout = 60,
		int rw_timeout = 60, bool unzip = true);
```
第一个是以已经连接成功的套接字流为参数的构造函数，该构造函数把连接 HTTP 服务器的工作交给用户来完成；第二个是以 HTTP 服务器地址为参数的构造函数，使用该构造函数，则该类对象内部会自动连接 HTTP 服务器。

下面的几个函数接口与 HTTP 发送相关：

```c++
	/**
	 * 获得 HTTP 请求头对象，然后在返回的 HTTP 请求头对象中添加
	 * 自己的请求头字段或 http_header::reset()重置请求头状态，
	 * 参考：http_header 类
	 * @return {http_header&}
	 */
	http_header& request_header(void);

	/**
	 * 向 HTTP 服务器发送 HTTP 请求头及 HTTP 请求体，同时从
	 * HTTP 服务器读取 HTTP 响应头，对于长连接，当连接中断时
	 * 会再重试一次，在调用下面的几个 get_body 函数前必须先
	 * 调用本函数(或调用 write_head/write_body)；
	 * 正常情况下，该函数在发送完请求数据后会读 HTTP 响应头，
	 * 所以用户在本函数返回 true 后可以调用：get_body() 或
	 * http_request::get_clinet()->read_body(char*, size_t)
	 * 继续读 HTTP 响应的数据体
	 * @param data {const void*} 发送的数据体地址，非空时自动按
	 *  POST 方法发送，否则按 GET 方法发送
	 * @param len {size_} data 非空时指定 data 数据长度
	 * @return {bool} 发送请求数据及读 HTTP 响应头数据是否成功
	 */
	bool request(const void* data, size_t len);

	/**
	 * 当采用流式写数据时，需要首先调用本函数发送 HTTP 请求头
	 * @return {bool} 是否成功，如果成功才可以继续调用 write_body
	 */
	bool write_head();

	/**
	 * 当采用流式写数据时，在调用 write_head 后，可以循环调用本函数
	 * 发送 HTTP 请求体数据；当输入的两个参数为空值时则表示数据写完；
	 * 当发送完数据后，该函数内部会自动读取 HTTP 响应头数据，用户可
	 * 继续调用 get_body/read_body 获取 HTTP 响应体数据
	 * @param data {const void*} 数据地址指针，当该值为空指针时表示
	 *  数据发送完毕
	 * @param len {size_t} data 非空指针时表示数据长度
	 * @return {bool} 发送数据体是否成功
	 *  注：当应用发送完数据后，必须再调用一次本函数，同时将两个参数都赋空
	 */
	bool write_body(const void* data, size_t len);
```

构建及发送 HTTP 请求的过程如下：

- 1、使用两个构造函数之一创建 acl::http_request 请求对象
- 2、调用 http_request::request_header 获得 HTTP 请求头对象的引用（http_header&），然后对该 HTTP 请求头设置 HTTP 请求的参数
- 3、http_request 类提供了两种 HTTP 请求调用 方式：
  - 3.1、当 HTTP 请求方法为 HTTP GET 方法或为 HTTP POST 但数据体可以一次性写入时，可以使用 http_request::request 方法，在调用 http_request::request 时会将 HTTP 请求头及请求体一次性发给 HTTP 服务器；
  - 3.2   如果为 HTTP POST 请求方法，且 HTTP 数据体内容是流式的（即每次只是要发送部分数据），则应该使用 http_request::write_head 和 http_request::write_body 两个函数，即使用流式方式发送数据时，应首先调用 http_request::write_head 发送 HTTP 请求头，当该函数返回成功后，可以循环调用 http_request::write_body 来发送 HTTP 请求数据体，为了表示 HTTP 请求体数据完毕，必须最后调用一次 http_request::write_body 且两个参数为 0 时以表示数据体发送完毕。

在调用以上 3.1 或 3.2 过程成功发送完 HTTP 请求数据后，这两个过程内部会自动读取 HTTP 服务器发来的 HTTP 响应头。

在上面的步骤 2 获得 HTTP 请求头对象（http_header）后，应该先调用下面的方法设置 HTTP 请求头中的参数：

```c++
	/**
	 * 设置请求的 URL，url 格式示例如下：
	 * 1、http://www.test.com/
	 * 2、/cgi-bin/test.cgi
	 * 3、http://www.test.com/cgi-bin/test.cgi
	 * 3、http://www.test.com/cgi-bin/test.cgi?name=value
	 * 4、/cgi-bin/test.cgi?name=value
	 * 5、http://www.test.com
	 * 如果该 url 中有主机字段，则内部自动添加主机；
	 * 如果该 url 中有参数字段，则内部自动进行处理并调用 add_param 方法；
	 * 调用该函数后用户仍可以调用 add_param 等函数添加其它参数；
	 * 当参数字段只有参数名没有参数值时，该参数将会被忽略，所以如果想
	 * 单独添加参数名，应该调用 add_param 方法来添加
	 * @param url {const char*} 请求的 url，非空指针
	 * @return {http_header&} 返回本对象的引用，便于用户连续操作
	 */
	http_header& set_url(const char* url);

	/**
	 * 设置 HTTP 请求头的 HOST 字段
	 * @param value {const char*} 请求头的 HOST 字段值
	 * @return {http_header&} 返回本对象的引用，便于用户连续操作
	 */
	http_header& set_host(const char* value);

	/**
	 * 向请求的 URL 中添加参数对，当只有参数名没有参数值时则：
	 * 1、参数名非空串，但参数值为空指针，则 URL 参数中只有：{name}
	 * 2、参数名非空串，但参数值为空串，则 URL参数中为：{name}=
	 * @param name {const char*} 参数名，不能为空指针
	 * @param value {const char*} 参数值，当为空指针时，仅添加参数名，
	 * @return {http_header&} 返回本对象的引用，便于用户连续操作
	 */
	http_header& add_param(const char* name, const char* value);
	http_header& add_int(const char* name, short value);
	http_header& add_int(const char* name, int value);
	http_header& add_int(const char* name, long value);
	http_header& add_int(const char* name, unsigned short value);
	http_header& add_int(const char* name, unsigned int value);
	http_header& add_int(const char* name, unsigned long value);
	http_header& add_format(const char* name, const char* fmt, ...)
		ACL_CPP_PRINTF(3, 4);

	/**
	 * 向 HTTP 头中添加 cookie
	 * @param name {const char*} cookie 名
	 * @param value {const char*} cookie 值
	 * @param domain {const char*} 所属域
	 * @param path {const char*} 存储路径
	 * @param expires {time_t} 过期时间，当该值为 0 时表示不过期，
	 *  > 0 时，则从现在起再增加 expires 即为过期时间，单位为秒
	 * @return {http_header&} 返回本对象的引用，便于用户连续操作
	 */
	http_header& add_cookie(const char* name, const char* value,
		const char* domain = NULL, const char* path = NULL,
		time_t expires = 0);

	/**
	 * 设置 HTTP 头中的 Connection 字段，是否保持长连接
	 * 不过，目前并未真正支持长连接，即使设置了该标志位，
	 * 则得到响应数据后也会主动关闭连接
	 * @param on {bool} 是否保持长连接
	 * @return {http_header&} 返回本对象的引用，便于用户连续操作
	 */
	http_header& set_keep_alive(bool on);

	/**
	 * 设置 HTTP 头中的 Content-Length 字段
	 * @param n {long long int} 设置值
	 * @return {http_header&} 返回本对象的引用，便于用户连续操作
	 */
	http_header& set_content_length(long long int n);

	/**
	 * 设置 HTTP 头中的 Content-Type 字段
	 * @param value {const char*} 设置值
	 * @return {http_header&} 返回本对象的引用，便于用户连续操作
	 */
	http_header& set_content_type(const char* value);
```

以上仅列出了 http_header 类设置 HTTP 请求参数的一些常用方法，其它的方法请参考 http_header.hpp 头文件中的说明。

## 二、acl::http_request 类获得 HTTP 服务器响应数据的常用方法
上面介绍了使用 acl::http_request 构建 HTTP 请求头及发送请求的接口方法，下面介绍使用 acl::http_request 类中的方法来接收 HTTP 服务器响应过程，在调用 http_request 类中的 request 或 write_body 成功发送完请求数据后，该类对象在这两个方法内部会首先自动接收 HTTP 服务器的响应头数据，若接收过程失败，这两个方法也会返回 false 表示失败，若返回成功，则可以调用 http_request 类对象的 http_status 方法获得 HTTP 服务器的响应状态码（2xx, 3xx, 4xx, 5xx），还可调用 body_length 方法获得 HTTP 响应数据体的长度（当 HTTP 服务器返回的数据格式为 HTTP 块传输时，该函数会返回 -1，所以一般不用显示调用该方法）。下面介绍了主要的与 HTTP 响应相关的方法：

首先是与 HTTP 响应头相关的接口函数，如下：

```c++
	/**
	 * 当发送完请求数据后，内部会自动调用读 HTTP 响应头过程，可以通过此函数获得服务端
	 * 响应的 HTTP 状态字(2xx, 3xx, 4xx, 5xx)；
	 * 其实该函数内部只是调用了 http_client::response_status 方法
	 * @return {int}
	 */
	int http_status() const;

	/**
	 * 获得 HTTP 响应的数据体长度
	 * @return {int64) 返回值若为 -1 则表明 HTTP 头不存在或没有长度字段
	 */
#ifdef WIN32
	__int64 body_length(void) const;
#else
	long long int body_length(void) const;
#endif
	/**
	 * HTTP 数据流(响应流是否允许保持长连接)
	 * @return {bool}
	 */
	bool keep_alive(void) const;

	/**
	 * 获得 HTTP 响应头中某个字段名的字段值
	 * @param name {const char*} 字段名
	 * @return {const char*} 字段值，为空时表示不存在
	 */
	const char* header_value(const char* name) const;

	/**
	 * 获得服务器返回的 Set-Cookie 设置的某个 cookie 对象
	 * @param name {const char*} cookie 名
	 * @param case_insensitive {bool} 是否区分大小写，true 表示
	 *  不区分大小写
	 * @return {const HttpCookie*} 返回 NULL 表示不存在
	 */
	const HttpCookie* get_cookie(const char* name,
		bool case_insensitive = true) const;
```

然后是与读 HTTP 响应数据体相关的接口函数：

```c++
	/**
	 * 是否读完了数据体
	 * @return {bool}
	 */
	bool body_finish() const;

	/**
	 * 当调用 request 成功后调用本函数，读取服务器响应体数据
	 * 并将结果存储于规定的 xml 对象中
	 * @param out {xml&} HTTP 响应体数据存储于该 xml 对象中
	 * @param to_charset {const char*} 当该项非空，内部自动
	 *  将数据转成该字符集存储于 xml 对象中
	 * @return {bool} 读数据是否成功
	 * 注：当响应数据体特别大时不应用此函数，以免内存耗光
	 */
	bool get_body(xml& out, const char* to_charset = NULL);

	/**
	 * 当调用 request 成功后调用本函数，读取服务器响应体数据
	 * 并将结果存储于规定的 json 对象中
	 * @param out {json&} HTTP 响应体数据存储于该 json 对象中
	 * @param to_charset {const char*} 当该项非空，内部自动
	 *  将数据转成该字符集存储于 json 对象中
	 * @return {bool} 读数据是否成功
	 * 注：当响应数据体特别大时不应用此函数，以免内存耗光
	 */
	bool get_body(json& out, const char* to_charset = NULL);

	/*
	 * 当调用 request 成功后调用本函数，读取服务器全部响应数据
	 * 存储于输入的缓冲区中
	 * @param out {string&} 存储响应数据体
	 * @param to_charset {const char*} 当该项非空，内部自动
	 *  将数据转成该字符集存储于 out 对象中
	 * 注：当响应数据体特别大时不应用此函数，以免内存耗光
	 */
	bool get_body(string& out, const char* to_charset = NULL);

	/*
	 * 当调用 request 成功后调用本函数，读取服务器响应数据并
	 * 存储于输入的缓冲区中，可以循环调用本函数，直至数据读完了，
	 * @param buf {char*} 存储部分响应数据体
	 * @param size {size_t} buf 缓冲区大小
	 * @return {int} 返回值 == 0 表示正常读完毕，< 0 表示服务器
	 *  关闭连接，> 0 表示已经读到的数据，用户应该一直读数据直到
	 *  返回值 <= 0 为止
	 *  注：该函数读到的是原始 HTTP 数据体数据，不做解压和字符集
	 *  解码，用户自己根据需要进行处理
	 */
	int read_body(char* buf, size_t size);

	/**
	 * 当调用 request 成功后调用本函数读 HTTP 响应数据体，可以循环调用
	 * 本函数，本函数内部自动对压缩数据进行解压，如果在调用本函数之前调用
	 * set_charset 设置了本地字符集，则还同时对数据进行字符集转码操作
	 * @param out {string&} 存储结果数据
	 * @param clean {bool} 每次调用本函数时，是否要求先自动将缓冲区 out
	 *  的数据清空
	 * @param real_size {int*} 当该指针非空时，存储解压前读到的真正数据
	 *  长度，如果在构造函数中指定了非自动解压模式且读到的数据 > 0，则该
	 *  值存储的长度值应该与本函数返回值相同；当读出错或未读到任何数据时，
	 *  该值存储的长度值为 0
	 * @return {int} == 0 表示读完毕，可能连接并未关闭；>0 表示本次读操作
	 *  读到的数据长度(当为解压后的数据时，则表示为解压之后的数据长度，
	 *  与真实读到的数据不同，真实读到的数据长度应该通过参数 real_size 来
	 *  获得); < 0 表示数据流关闭，此时若 real_size 非空，则 real_size 存
	 *  储的值应该为 0
	 */
	int read_body(string& out, bool clean = false, int* real_size = NULL);

	/**
	 * 当调用 request 成功后调用本函数来从 HTTP 服务端读一行数据，可以循环调用
	 * 本函数，直到返回 false 或 body_finish() 返回 true 为止；
	 * 本函数内部自动对压缩数据进行解压，如果在调用本函数之前调用 set_charset 设置了
	 * 本地字符集，则还同时对数据进行字符集转码操作
	 * @param out {string&} 存储结果数据
	 * @param nonl {bool} 读到的一行数据是否自动去掉尾部的 "\r\n" 或 "\n"
	 * @param size {size_t*} 该指针非空时存放读到的数据长度
	 * @return {bool} 是否读到了一行数据：当返回 true 时表示读到了一行数据，可以
	 *  通过 body_finish() 是否为 true 来判断是否读数据体已经结束，当读到一个空行
	 *  且 nonl = true 时，则 *size = 0；当返回 false 时表示未读完整行且读完毕，
	 *  *size 中存放着读到的数据长度
	 */
	bool body_gets(string& out, bool nonl = true, size_t* size = NULL);
```

虽然上面提供了多个读 HTTP 响应体数据的方法，但可以分为两大类：1、一次性读所有的数据体；2、以流式方式循环读数据体。 其中，对于“一次性读取所有数据体”的读方法，适合于响应数据体比较小的情形，当响应数据为 xml 或 json 格式时，还提供了直接将响应数据体转为 xml 或 json 对象的读方法；如果响应数据体非常大（如几兆甚至几十兆以上）则应该采用流式方法循环读数据体。

有一点需要注意，除了 " int read_body(char* buf, size_t size);" 可以直接读原生的响应数据体外，其它的读方法会将读到数据体自动进行解压、字符集转换操作后将最终结果返回调用者。

此外，为了方便一些文本类应用，在 http_request 类中还提供了 body_gets 方法，用来以行为单位读取 HTTP 响应数据体（当服务器也是以行为单位发送响应数据时才可使用 body_gets 方法）。

acl::http_request 类除了以上接口外，还提供了其它丰富的接口（如：支持 HTTP 断点续传的 Range 相关的方法），如果您觉得这些接口依然不能满足要求，不妨通过 "http_request::get_client" 获得 acl::http_client 类对象（该类对象是 acl 有关 http 协议处理中比较基础的 HTTP 通信类），然后再在 acl::http_client 类中查找您所希望的功能接口。

## 三、示例
下面用一个简单的例子来说明上面一些方法的使用过程：

```c++
// http_servlet.cpp : 定义控制台应用程序的入口点。
//
#include <assert.h>
#include <getopt.h>
#include "acl_cpp/lib_acl.hpp"

using namespace acl;

//////////////////////////////////////////////////////////////////////////

class http_request_test
{
public:
	http_request_test(const char* server_addr, const char* file,
		const char* stype, const char* charset)
	{
		server_addr_= server_addr;
		file_ = file;
		stype_ = stype;
		charset_ = charset;
		to_charset_ = "gb2312";
	}

	~http_request_test() {}

	bool run(void)
	{
		string body;
		if (ifstream::load(file_, &body) == false)
		{
			logger_error("load %s error", file_.c_str());
			return false;
		}

		http_request req(server_addr_);

		// 添加 HTTP 请求头字段

		string ctype("text/");
		ctype << stype_ << "; charset=" << charset_;

		http_header& hdr = req.request_header();  // 请求头对象的引用
		hdr.set_url("/");
		hdr.set_content_type(ctype);
		hdr.add_param("name1", "value1");
		hdr.add_param("name2", "value2");
		// 发送 HTTP 请求数据
		if (req.request(body.c_str(), body.length()) == false)
		{
			logger_error("send http request to %s error",
				server_addr_.c_str());
			return false;
		}

		// 取出 HTTP 响应头的 Content-Type 字段

		const char* p = req.header_value("Content-Type");
		if (p == NULL || *p == 0)
		{
			logger_error("no Content-Type");
			return false;
		}

		// 分析 HTTP 响应头的数据类型
		http_ctype content_type;
		content_type.parse(p);

		// 响应头数据类型的子类型
		const char* stype = content_type.get_stype();

		bool ret;
		if (stype == NULL)
			ret = do_plain(req);
		else if (strcasecmp(stype, "xml") == 0)
			ret = do_xml(req);
		else if (strcasecmp(stype, "json") == 0)
			ret = do_json(req);
		else
			ret = do_plain(req);
		if (ret == true)
			logger("read ok!\r\n");
		return ret;
	}

private:
	// 处理 text/plain 类型数据
	bool do_plain(http_request& req)
	{
		string body;
		if (req.get_body(body, to_charset_) == false)
		{
			logger_error("get http body error");
			return false;
		}
		printf("body:\r\n(%s)\r\n", body.c_str());
		return true;
	}

	// 处理 text/xml 类型数据
	bool do_xml(http_request& req)
	{
		xml body;
		if (req.get_body(body, to_charset_) == false)
		{
			logger_error("get http body error");
			return false;
		}
		xml_node* node = body.first_node();
		while (node)
		{
			const char* tag = node->tag_name();
			const char* name = node->attr_value("name");
			const char* pass = node->attr_value("pass");
			printf(">>tag: %s, name: %s, pass: %s\r\n",
				tag ? tag : "null",
				name ? name : "null",
				pass ? pass : "null");
			node = body.next_node();
		}
		return true;
	}

	// 处理 text/json 类型数据
	bool do_json(http_request& req)
	{
		json body;
		if (req.get_body(body, to_charset_) == false)
		{
			logger_error("get http body error");
			return false;
		}

		json_node* node = body.first_node();
		while (node)
		{
			if (node->tag_name())
			{
				printf("tag: %s", node->tag_name());
				if (node->get_text())
					printf(", value: %s\r\n", node->get_text());
				else
					printf("\r\n");
			}
			node = body.next_node();
		}
		return true;
	}

private:
	string server_addr_;	// web 服务器地址
	string file_;		// 本地请求的数据文件
	string stype_;		// 请求数据的子数据类型
	string charset_;	// 本地请求数据文件的字符集
	string to_charset_;	// 将服务器响应数据转为本地字符集
};

//////////////////////////////////////////////////////////////////////////

static void usage(const char* procname)
{
	printf("usage: %s -h[help]\r\n", procname);
	printf("options:\r\n");
	printf("\t-f request file\r\n");
	printf("\t-t request stype[xml/json/plain]\r\n");
	printf("\t-c request file's charset[gb2312/utf-8]\r\n");
}

int main(int argc, char* argv[])
{
	int   ch;
	string server_addr("127.0.0.1:8888"), file("./xml.txt");
	string stype("xml"), charset("gb2312");

	while ((ch = getopt(argc, argv, "hs:f:t:c:")) > 0)
	{
		switch (ch)
		{
		case 'h':
			usage(argv[0]);
			return 0;
		case 'f':
			file = optarg;
			break;
		case 't':
			stype = optarg;
			break;
		case 'c':
			charset = optarg;
			break;
		default:
			usage(argv[0]);
			return 0;
		}
	}

	log::stdout_open(true);   // 允许日志输出至屏幕上
	http_request_test test(server_addr, file, stype, charset);
	test.run();  // 开始运行

	return 0;
} 
```

上面的例子来自于 lib_acl_cpp/samples/http_request。

如果查看 http_request::request 源码实现，会发现 try_open()、reuse_conn、need_retry_ 等方法或变量来表示 HTTP 客户端连接的重试过程，这是因为 http_request 类的设计是支持长连接及可重用的，对于 HTTP 客户端连接池来说这些功能非常重要，在下一节介绍使用 acl 的 http 客户端连接池功能类时将会用到 http 请求客户端连接的重连及重试机制。

## 四、参考

http_request 类的头文件位置：lib_acl_cpp/include/acl_cpp/http/http_request.hpp
github：https://github.com/acl-dev/acl
gitee：https://gitee.com/acl-dev/acl