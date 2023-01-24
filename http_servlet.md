---
title: 用C++实现类似于JAVA HttpServlet 的编程接口
date: 2012-05-20 13:08:24
tags: http
categories: http开发
---

## 一、概述
互联网刚兴起时，很多项目都是用 C /Perl 语言写的一大堆 CGI，一些老程序员可谓是偿尽了编程的苦，因为那时国内的技术水平普遍比较低，如果你会 CGI 编程，就已经算是行业中人了，如果你对 CGI 编程比较熟练，则就可以称得是“专家”了，后来技术不断进步，各种国外的新技术都进入中国并不断得到普及，CGI 就逐渐沦为一种落后的技术，后来的 PHP, JSP/Servlet, ASP 逐渐占领了 WEB 编程的技术市场，这个时候如果你说再用 C 写 CGI，别人会感觉是在和古人对话。现在主流的 WEB 开发语言一个很大的优势就是有各种相对成熟的基础库和框架，开发效率很高，而 CGI 则就逊色很多。当然，这些语言也得有执行效率相对较低的问题，毕竟它们都是脚本语言或半编译语言，需要虚拟机解释执行，象 facebook 的 WEB 前端基本都是用 PHP 写的，他们为了解决执行效率问题，在一位华人的领导下开发了可以将 PHP 代码转成 C++ 代码的工具（hiphop)，从而使执行效率大大提高，这也从另一个侧面反映出技术人员还是希望他们的程序能够运行的更快些。

本文主要描述了 acl_cpp 库中有关 WEB 编程的方法类，为了使大家容易上手，其中的接口设计及命名尽量模仿 JAVA HttpServlet 等相关的类（希望 Oracle 不会告我侵权，呵呵）。如果您会用C/C++编程，同时又有使用 Java Servlet 进行 WEB 编程的经验，则该文您读起来一点不会费力，当然如果您多年从事 WEB 开发，我想理解这些类的设计及用法也不应该有什么难度。好了，下面就开始讲如何使用 acl_cpp 库中的 http/ 模块下的类进行 web 编程。

在 acl_cpp/src/http 模块下，有几个类与 WEB 编程相关：HttpServlet，HttpServletRequest, HttpServletResponse, HttpSession, http_header, http_mime, http_client。如果您掌握了这几个类的用法，则进行 WEB 编程就不会有什么问题了，下面一一介绍这几个类：

## 二、HttpServlet 类
构造函数及析构函数：
```c++
	/**
	 * 构造函数
	 */
	HttpServlet(void);

	/**
	 * 纯虚析构函数，即该类必须由子类进行实例化
	 */
	virtual ~HttpServlet(void) = 0;
```

在构建函数中，为了支持 HttpSession 数据的存储，需要用户给出 memcached 的服务器地址（目前仅支持采用 memcached 来存储 session 数据，将来应该会扩展至可以支持 redis 等），同时用户还需要给出 session 的 cookie ID 标识符以发给浏览器。

四个虚接口，需要子类实现以应对不同的浏览器的 HTTP 请求：
```c++
	/**
	 * 当 HTTP 请求为 GET 方式时的虚函数
	 */
	virtual bool doGet(HttpServletRequest&, HttpServletResponse&);

	/**
	 * 当 HTTP 请求为 POST 方式时的虚函数
	 */
	virtual bool doPost(HttpServletRequest&, HttpServletResponse&);

	/**
	 * 当 HTTP 请求为 PUT 方式时的虚函数
	 */
	virtual bool doPut(HttpServletRequest&, HttpServletResponse&);

	/**
	 * 当 HTTP 请求为 CONNECT 方式时的虚函数
	 */
	virtual bool doConnect(HttpServletRequest&, HttpServletResponse&);

	/**
	 * 当 HTTP 请求为 PURGE 方式时的虚函数，该方法在清除 SQUID 的缓存
	 * 时会用到
	 */
	virtual bool doPurge(HttpServletRequest&, HttpServletResponse&);
```

用户实现的 HttpServlet 子类中可以实现以上几个虚接口的一个或者几个，以满足不同的 HTTP 请求。
 
下面的函数为 HttpServlet 类开始运行的函数：
```c++
	/**
	 * HttpServlet 对象开始运行，接收 HTTP 请求，并回调以下 doXXX 虚函数
	 * @param session {session&} 存储 session 数据的对象
	 * @param stream {socket_stream*} 当在 acl_master 服务器框架控制下
	 *  运行时，该参数必须非空；当在 apache 下以 CGI 方式运行时，该参数
	 *  设为 NULL；另外，该函数内部不会关闭流连接，应用应自行处理流对象
	 *  的关闭情况，这样可以方便与 acl_master 架构结合
	 * @param body_parse {bool} 针对 POST 方法，该参数指定是否需要
	 *  读取 HTTP 请求数据体并按 n/v 方式进行分析；当为 true 则内
	 *  部会读取 HTTP 请求体数据，并进行分析，当用户调用 getParameter
	 *  时，不仅可以获得 URL 中的参数，同时可以获得 POST 数据体中
	 *  的参数；当该参数为 false 时则不读取数据体
	 * @param body_limit {int} 针对 POST 方法，当数据体为文本参数
	 *  类型时，此参数限制数据体的长度；当数据体为数据流或 MIME
	 *  格式或 body_read 为 false，此参数无效
	 * @return {bool} 返回处理结果
	 */
	bool doRun(session& session, socket_stream* stream = NULL,
		bool body_parse = true, int body_limit = 102400);

	/**
	 * HttpServlet 对象开始运行，接收 HTTP 请求，并回调以下 doXXX 虚函数，
	 * 调用本函数意味着采用 memcached 来存储 session 数据
	 * @param memcached_addr {const char*} memcached 服务器地址，格式：IP:PORT
	 * @param stream {socket_stream*} 含义同上
	 * @param body_parse {bool} 含义同上
	 * @param body_limit {int} 含义同上
	 * @return {bool} 返回处理结果
	 */
	bool doRun(const char* memcached_addr = "127.0.0.1:11211",
		socket_stream* stream = NULL,
		bool body_parse = true, int body_limit = 102400);

```

从上面五个虚方法中，可以看到两个重要的类：HttpServletRequest 和 HttpServletResponse。这两个类分别表示 http 请求类及 http 响应类，这两个类都是由 HttpServlet 类对象创建并释放的，所以用户不必创建和销毁这两个类对象实例。下面分别介绍这两个类：

## 三、HttpServletRequest 类
该类主要是与浏览器的请求过程相关，您可以通过该类的方法获得浏览器的请求数据。该类的方法比较多（基本上是参照了 java HttpServlet 的功能方法及名称），所以下面仅介绍几个主要的方法：

```c++
	/**
	 * 获得 HTTP 请求中的参数值，该值已经被 URL 解码且
	 * 转换成本地要求的字符集；针对 GET 方法，则是获得
	 * URL 中 ? 后面的参数值；针对 POST 方法，则可以获得
	 * URL 中 ? 后面的参数值或请求体中的参数值
	 */
	const char* getParameter(const char* name) const;

	/**
	 * 获得 HTTP 客户端请求的某个 cookie 值
	 * @param name {const char*} cookie 名称，必须非空
	 * @return {const char*} cookie 值，当返回 NULL 时表示
	 *  cookie 值不存在
	 */
	const char* getCookieValue(const char* name) const;

	/**
	 * 获得与该 HTTP 会话相关的 HttpSession 对象引用
	 * @return {HttpSession&}
	 */
	HttpSession& getSession(void);

	/**
	 * 获得与 HTTP 客户端连接关联的输入流对象引用
	 * @return {istream&}
	 */
	istream& getInputStream(void) const;

	/**
	 * 获得 HTTP 请求数据的数据长度
	 * @return {acl_int64} 返回 -1 表示可能为 GET 方法，
	 *  或 HTTP 请求头中没有 Content-Length 字段
	 */
#ifdef WIN32
	__int64 getContentLength(void) const;
#else
	long long int getContentLength(void) const;
#endif

	/**
	 * 当 HTTP 请求头中的 Content-Type 为
	 * multipart/form-data; boundary=xxx 格式时，说明为文件上传
	 * 数据类型，则可以通过此函数获得 http_mime 对象
	 * @return {const http_mime*} 返回 NULL 则说明没有 MIME 对象，
	 *  返回的值用户不能手工释放，因为在 HttpServletRequest 的析
	 *  构中会自动释放
	 */
	http_mime* getHttpMime(void) const;

	/**
	 * 获得 HTTP 请求数据的类型
	 * @return {http_request_t}，一般对 POST 方法中的上传
	 *  文件应用而言，需要调用该函数获得是否是上传数据类型
	 */
	http_request_t getRequestType(void) const;
```

以上方法一般都是我们在实际对 HttpServletRequest 类方法使用过程中用得较多的。如：

```
    getParmeter： 用来获得 http 请求参数
    getCookieValue：获得浏览器的 cookie 值
    getSession：获得该 HttpServlet 类对象的 session 会话
    getInputStream：获得 http 连接的输入流
    getContentLength：针对 HTTP POST 请求，此函数获得 HTTP 请求数据体的长度
    getRequestType：针对 HTTP POST 请求，此函数返回 HTTP 请求数据体的传输方式（普通的 name=value 方式，multipart 上传文件格式以及数据流格式）
```
 
## 四、HttpServletResponse 类
该类主要与将您写的程序将处理数据结果返回给浏览器的过程相关，下面也仅介绍该类的一些常用的函数，如果您需要更多的功能，请参数 HttpServletResponse.hpp 头文件。

```c++
	/**
	 * 设置 HTTP 响应数据体的 Content-Type 字段值，可字段值可以为：
	 * text/html 或 text/html; charset=utf8 格式
	 * @param value {const char*} 字段值
	 */
	void setContentType(const char* value);

	/**
	 * 设置 HTTP 响应数据体中字符集，当已经在 setContentType 设置
	 * 了字符集，则就不必再调用本函数设置字符集
	 * @param charset {const char*} 响应体数据的字符集
	 */
	void setCharacterEncoding(const char* charset);

	/**
	 * 设置 HTTP 响应头中的状态码：1xx, 2xx, 3xx, 4xx, 5xx
	 * @param status {int} HTTP 响应状态码, 如：200
	 */
	void setStatus(int status);

	/**
	 * 添加 cookie
	 * @param name {const char*} cookie 名
	 * @param value {const char*} cookie 值
	 * @param domain {const char*} cookie 存储域
	 * @param path {const char*} cookie 存储路径
	 * @param expires {time_t} cookie 过期时间间隔，当当前时间加
	 *  该值为 cookie 的过期时间截(秒)
	 */
	void addCookie(const char* name, const char* value,
		const char* domain = NULL, const char* path = NULL, time_t expires = 0);

	/**
	 * 发送 HTTP 响应头，用户应该发送数据体前调用此函数将 HTTP
	 * 响应头发送给客户端
	 */
	bool sendHeader(void);

	/**
	 * 获得 HTTP 响应对象的输出流对象，用户在调用 sendHeader 发送
	 * 完 HTTP 响应头后，通过该输出流来发送 HTTP 数据体
	 * @return {ostream&}
	 */
	ostream& getOutputStream(void) const;
```

- setCharacterEncoding：该方法设置 HTTP 响应头的 HTTP 数据体的字符集，如果通过该函数设置了字符集，即使您在返回的 html 数据中重新设置了其它的字符集，浏览器也会优先使用 HTTP 响应头中设置的字符集，所以用户一定得注意这点；
- setContentType：该方法用来设置 HTTP 响应头中的 Content-Type 字段，对于 xml 数据则设置 text/xml，对 html 数据则设置 text/html，当然您也可以设置 image/jpeg 等数据类型；当然，您也可以直接通过该方法在设置数据类型的同时指定数据的字符集，如可以直接写：setContentType("text/html; charset=utf8")，这个用法等同于：setContentType("text/html"); setCharacterEncoding("utf8")。
- setStatus：设置 HTTP 响应头的状态码（一般不用设置状态码，除非是您确实需要单独设置）；
- addCookie：在 HTTP 响应头中添加 cookie 内容；
- sendHeader：发送 HTTP 响应头；
- getOutputStream：该函数返回输出流对象，您可以向输出流中直接写 HTTP 响应的数据体（关于 ostream 类的使用请参数头文件：include/ostream.hpp）。

除了以上三个类外，还有一个类比较重要：HttpSession 类，该类主要实现与 session 会话相关的功能：

## 五、HttpSession 类
该类对象实例用户也不必创建与释放，在 HttpServet 类对象内容自动管理该类对象实例。主要用的方法有：
```c++
	/**
	 * 获得客户端在服务端存储的对应 session 变量名，子类可以重载该方法
	 * @param name {const char*} session 名，非空
	 * @return {const char*} session 值，为空说明不存在或内部
	 *  查询失败
	 */
	virtual const char* getAttribute(const char* name) const;

	/**
	 * 设置服务端对应 session 名的 session 值，子类可以重载该方法
	 * @param name {const char*} session 名，非空
	 * @param name {const char*} session 值，非空
	 * @return {bool} 返回 false 说明设置失败
	 */
	virtual bool setAttribute(const char* name, const char* value);
```

只所以将这两个方法声明为虚方法，是因为 HttpSession 的 session 数据存储目前仅支持 memcached，您如果有精力的话可以实现一个子类用来支持其它的数据存储方式。当然您也可以在您实现的子类中实现自己的产生唯一 session id 的方法，即实现如下虚方法：

```c++
protected:
	/**
	 * 创建某个 session 会话的唯一 ID 号，子类可以重载该方法
	 * @param buf {char*} 存储结果缓冲区
	 * @param len {size_t} buf 缓冲区大小，buf 缓冲区大小建议
	 *  64 字节左右
	 */
	virtual void createSid(char* buf, size_t len);
```

好了，上面说了一大堆类及类函数，下面还是以一个具体的示例来说明这些类的用法：

## 六、示例
下面的例子是一个 CGI 例子，编译后可执行程序可以直接放在 apache 的 cgi-bin/ 目录，用户可以用浏览器访问。

```c++
// http_servlet.cpp : 定义控制台应用程序的入口点。
//

#include "lib_acl.hpp"
using namespace acl;

//////////////////////////////////////////////////////////////////////////

class http_servlet : public HttpServlet
{
public:
	http_servlet(void)
	{
	}

	~http_servlet(void)
	{
	}

	// 实现处理 HTTP GET 请求的功能函数
	bool doGet(HttpServletRequest& req, HttpServletResponse& res)
	{
		return doPost(req, res);
	}

	// 实现处理 HTTP POST 请求的功能函数
	bool doPost(HttpServletRequest& req, HttpServletResponse& res)
	{
		// 获得某浏览器用户的 session 的某个变量值，如果不存在则设置一个
		const char* sid = req.getSession().getAttribute("sid");
		if (sid == NULL || *sid == 0)
			req.getSession().setAttribute("sid", "xxxxxx");

		// 再取一次该浏览器用户的 session 的某个属性值
		sid = req.getSession().getAttribute("sid");

		// 取得浏览器发来的两个 cookie 值
		const char* cookie1 = req.getCookieValue("name1");
		const char* cookie2 = req.getCookieValue("name2");

		// 开始创建 HTTP 响应头

		// 设置 cookie
  		res.addCookie("name1", "value1");

		// 设置具有作用域和过期时间的 cookie
		res.addCookie("name2", "value2", ".test.com", "/", 3600 * 24);
		// res.setStatus(200);  // 可以设置返回的状态码

		// 两种方式都可以设置字符集
		if (0)
			res.setContentType("text/xml; charset=gb2312");
		else {
			// 先设置数据类型
			res.setContentType("text/xml");

			// 再设置数据字符集
			res.setCharacterEncoding("gb2312");
		}

		// 获得浏览器请求的两个参数值
		const char* param1 = req.getParameter("name1");
		const char* param2 = req.getParameter("name2");

		// 创建 xml 格式的数据体
		xml body;
		body.get_root().add_child("root", true)
			.add_child("sessions", true)
				.add_child("session", true)
					.add_attr("sid", sid ? sid : "null")
					.get_parent()
				.get_parent()
			.add_child("cookies", true)
				.add_child("cookie", true)
					.add_attr("name1", cookie1 ? cookie1 : "null")
					.get_parent()
				.add_child("cookie", true)
					.add_attr("name2", cookie2 ? cookie2 : "null")
					.get_parent()
				.get_parent()
			.add_child("params", true)
				.add_child("param", true)
					.add_attr("name1", param1 ? param1 : "null")
					.get_parent()
				.add_child("param", true)
					.add_attr("name2", param2 ? param2 : "null");
		string buf;
		body.build_xml(buf);
		// 在http 响应头中设置数据体长度
		res.setContentLength(buf.length());
		// 发送 http 响应头
		if (res.sendHeader() == false)
			return false;
		// 发送 http 响应体
		if (res.write(buf) == false)
			return false;
		return true;
	}
};

//////////////////////////////////////////////////////////////////////////

int main(void)
{
#ifdef WIN32
	acl::acl_cpp_init();  // win32 环境下需要初始化库
#endif

	http_servlet servle;

	// cgi 开始运行
	servlet.doRun("127.0.0.1:11211");  // 开始运行，并设定 memcached 的服务地址为：127.0.0.1:11211

	// 运行完毕，程序退出
	return 0;
}
```

经常使用 Java HttpServlet 等类进行 web 编程的用户对上面的代码一定不会感到陌生，但它的的确确是一个 CGI 程序，可以放在 Apache 及支持 CGI 的 Webserver 下运行。当然，大家应该都清楚 CGI 在运行时因进程切换而导致了效率较为低下，在另一篇文章《使用 acl_cpp 的 HttpServlet 类及服务器框架编写WEB服务器程序 》中展示了用上面的 http_servlet 类并结合 acl_cpp 的服务器模型实现的一个WEB服务器的例子，效率比 CGI 要高的多(效率也应比 FCGI高，因为其少了 Webserver 层的过滤)；文章《acl_cpp web 编程之文件上传 》中举例讲述了在服务端如何使用 acl_cpp 库处理浏览器上传文件的功能。

acl 下载
github: https://github.com/acl-dev/acl
gitee: https://github.com/acl-dev/acl
