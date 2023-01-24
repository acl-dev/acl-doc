---
title: Web 编程中实现文件上传的服务端实例
date: 2012-05-22 13:08:24
tags: http
categories: http开发
---

在文章《用C++实现类似于JAVA HttpServlet 的编程接口 》中讲了如何用 acl_cpp 的 HttpServlet 等类来实现 WEB CGI 的功能，同时在文章《使用 acl_cpp 的 HttpServlet 类及服务器框架编写WEB服务器程序 》中也举例说明如何将基于 HttpServlet 编写的 CGI 程序快速地转为服务器程序的过程。本文主要讲如何用 acl_cpp 的 WEB 编程类实现 HTTP 文件上传过程。为了实现 HTTP 协议的文件上传过程，引入了两个类：http_mime 和 http_mime_node。

http_mime 类是有关 HTTP 协议中 mime 格式的流式解析器（即每次仅输入部分 HTTP MIME 数据，等数据输入完毕时，该解析器也解析完毕，流式解析的好处是它可以适用于阻塞或非阻塞的IO模式）；http_mime_node 类对象表示 http mime 数据中每一个 mime 结点对象，该结点的数据可能是文件内容数据，也可能是参数数据。

## 一、http_mime 类
该类一般由 HttpServletRequest 类内部自动管理（负责分配与释放 http_mide 类对象），当然用户可以在测试 http_mime 类时，自己创建与释放该类对象。下面是该类的构造函数及常用方法：

```c++
	/**
	 * 构建函数
	 * @param boundary {const char*} 分隔符，不能为空
	 * @param local_charset {const char*} 本地字符集，非空时会自动将
	 *  参数内容转为本地字符集
	 */
	http_mime(const char* boundary, const char* local_charset  = "gb2312");
```

尤其需要指出的是 http mime 的 boundary(分隔符）与邮件的 mime 的分隔符规则略有不同，如邮件的相关头部字段为：Content-Type: multipart/mixed; charset="GB2312"; boundary="0_11119_1331286082"，HTTP MIME 的相关头部字段为：Content-Type: multipart/form-data; boundary="--0_11119_1331286082"。其中，最大的区别就是在 HTTP 头中获得的分隔符与 HTTP 数据体的分隔符（除结尾分隔符多了两个 '-' 后缀）完全相同，而邮件的 mime 的分隔符在头部和 mime 体中是不一样的，mime 体中的分隔符是由头部的分隔符加两个 '-' 作为前导符（结尾分隔符为头部分隔符前面加两个 '-'，尾部加两个 '-'），一定得注意这些不同。在 acl_cpp 中的 http mime 解析模块原来主要是作邮件 mime 解析的，现在依然支持 HTTP 的 mime 解析，唯一不同就是区分分隔符的不同。（当然，邮件的 MIME 数据体还与 HTTP MIME 数据体有另外一个区别：邮件的 MIME 数据一般都是要经过 BASE64 来编码的，而 HTTP MIME 却很少编码）。

http_mime 的几个常用方法接口如下：
```c++
	/**
	 * 设置 MIME 数据的存储路径，当分析完 MIME 数据后，如果想要从中提取数据，
     * 则必须给出该 MIME 的原始数据的存储位置，否则无法获得相应数据，即
     * save_xxx/get_nodes/get_node 函数均无法正常使用
	 * @param path {const char*} 文件路径名, 如果该参数为空, 则不能
	 *  获得数据体数据, 也不能调用 save_xxx 相关的接口
	 */
	void set_saved_path(const char* path);

	/**
	 * 调用此函数进行流式方式解析数据体内容
	 * @param data {const char*} 数据体(可能是数据头也可能是数据体, 
	 *  并且不必是完整的数据行)
	 * @param len {size_t} data 数据长度
	 * @return {bool} 针对 multipart 数据, 返回 true 表示解析完毕;
	 *  对于非 multipart 文件, 该返回值永远为 false, 没有任何意义, 
	 *  需要调用者自己判断数据体的结束位置
	 * 注意: 调用完此函数后一定需要调用 update_end 函数通知解析器解析完毕
	 */
	bool update(const char* data, size_t len);

	/**
	 * 获得所有的 MIME 结点
	 * @return {const std::list<http_mimde_node*>&}
	 */
	const std::list<http_mime_node*>& get_nodes(void) const;

	/**
	 * 根据变量名取得 HTTP MIME 结点
	 * @param param name {const char*} 变量名
	 * @return {http_mime_node*} 返回空则说明对应变量名的结点不存在
	 */
	const http_mime_node* get_node(const char* name) const;
```

## 二、http_mime_node 类
该类实例存储 HTTP MIME 数据体中每个数据结点，同时该类的实例是由 http_mime 类对象自动维护的，所以您一般不必关心该类对象的创建与销毁；另外，http_mime_node 类的继承关系为：http_mime_node -> mime_attach -> mime_node。

该类的构造函数如下：
```c++
	/**
	 * 原始文件存放路径，不能为空
	 * @param node {MIME_NODE*} 对应的 MIME 结点，非空
	 * @param decodeIt {bool} 是否对 MIME 结点的头部数据
	 *  或数据体数据进行解码
	 * @param toCharset {const char*} 本机的字符集
	 * @param off {off_t} 偏移数据位置
	 */
	http_mime_node(const char* path, const MIME_NODE* node,
		bool decodeIt = true, const char* toCharset = "gb2312", off_t off = 0);
```

 该类的常用方法为：
```c++
	/**
	 * 获得该结点的类型
	 * @return {http_mime_t}
	 */
	http_mime_t get_mime_type(void) const;

	/**
	 * 当 get_mime_type 返回的类型为 HTTP_MIME_PARAM 时，可以
	 * 调用此函数获得参数值；参数名可以通过基类的 get_name() 获得
	 * @return {const char*} 返回 NULL 表示参数不存在
	 */
	const char* get_value(void) const;
```

http_mime_t 为枚举类型，如：
```c++
typedef enum
{
	HTTP_MIME_PARAM,        // http mime 结点为参数类型
	HTTP_MIME_FILE          // http mime 结点为文件类型
} http_mime_t;
```
加上两个基类的一些方法，有几个方法也是比较常用的，如下：
- mime_node::get_name: 获得该 mime 结点的名称
- mime_attach::get_filename: 当结点为上传文件类型时，此函数获得上传文件的文件名
## 三、示例
```c++
#include "lib_acl.hpp"

using namespace acl;

class http_servlet : public HttpServlet
{
public:
	http_servlet()
	{
		...
	}

	...
	// 基类虚方法：HTTP POST 方法接口
	bool doPost(HttpServletRequest& req, HttpServletResponse& res)
	{
		...
		return doUpload(req, res);
	}

	// 处理文件上传的函数
	bool doUpload(HttpServletRequest& req, HttpServletResponse& res)
	{
		// 先获得 Content-Type 对应的 http_ctype 对象
		http_mime* mime = req.getHttpMime();
		if (mime == NULL)
		{
			logger_error("http_mime null");
			return false;
		}

		// 获得数据体的长度
		long long int len = req.getContentLength();
		if (len <= 0)
		{
			logger_error("body empty");
			return false;
		}

		// 获得输入流
		istream& in = req.getInputStream();
		char  buf[8192];
		int   ret;
		bool  n = false;

		const char* filepath = "./var/mime_file";
		ofstream out;
		// 只写方式打开存储上传文件的临时文件句柄
		out.open_write(filepath);

		// 设置原始文件存入路径
		mime->set_saved_path(filepath);

		// 读取 HTTP 客户端请求数据
		while (len > 0)
		{
			// 从 HTTP 输入流中读取数据
			ret = in.read(buf, sizeof(buf), false);
			if (ret == -1)
			{
				logger_error("read POST data error");
				return false;
			}
			// 将数据写入临时文件中
			out.write(buf, ret);
			len -= ret;

			// 将读得到的数据输入至解析器进行解析
			if (mime->update(buf, ret) == true)
			{
				n = true;
				break;
			}
		}
		out.close();

		if (len != 0 || n == false)
			logger_warn("not read all data from client");

		string path;

		// 遍历所有的 MIME 结点，找出其中为文件结点的部分进行转储
		const std::list<http_mime_node*>& nodes = mime->get_nodes();
		std::list<http_mime_node*>::const_iterator cit = nodes.begin();
		for (; cit != nodes.end(); ++cit)
		{
			// HTTP MIME 结点的变量名
			const char* name = (*cit)->get_name();

			// HTTP MIME 结点的类型
			http_mime_t mime_type = (*cit)->get_mime_type();
			if (mime_type == HTTP_MIME_FILE)
			{
				// 当该结点为文件数据结点时
				// 取得上传文件名
				const char* filename = (*cit)->get_filename();
				if (filename == NULL)
				{
					logger("filename null");
					continue;
				}

				if (strcmp(name, "file1") == 0)
					file1_ = filename;
				else if (strcmp(name, "file2") == 0)
					file2_ = filename;
				else if (strcmp(name, "file3") == 0)
					file3_ = filename;

				// 将文件内容转存
				path.format("./var/%s", filename);
				(void) (*cit)->save(path.c_str());
			}
		}

		// 查找上载的某个文件并转储
		const http_mime_node* node = mime->get_node("file1");
		if (node && node->get_mime_type() == HTTP_MIME_FILE)
		{
			const char* ptr = node->get_filename();
			if (ptr)
			{
				path.format("./var/1_%s", ptr);
				(void) node->save(path.c_str());
			}
		}

		// 删除临时文件
		:unlink(filepath);

		// 发送 http 响应头
		if (res.sendHeader() == false)
			return false;
		// 发送 http 响应体
		if (res.getOutputStream().write("ok") == -1)
			return false;
		return true;
	}

private:
	const char* file1_;
	const char* file2_;
	const char* file3_;
};

int main(void)
{
#ifdef WIN32
	acl::acl_cpp_init();
#endif

	// 开始运行
	http_servlet servlet;
	servlet.doRun("127.0.0.1:11211"); // 开始运行，并假设 memcached 监听于 127.0.0.1:11211
	return 0;
}
```

与上面例子对应的 HTML 页面如下：

```html
<html>
<head>
<meta content="text/html; charset=gb2312" http-equiv="Content-Type">
</head>
<body>
<form enctype="multipart/form-data" method=POST action="/cgi-bin/test/upload?name1=中国人">
<input type=hidden name="name2" value="美国人"><br>
<input type=hidden name="name3" value="英国人"><br>
<input type=submit name="submit", value="提交"><br>
文件一：<input type=file name="file1" value=""><br>
文件二：<input type=file name="file2" value=""><br>
文件三：<input type=file name="file3" value=""><br>
</form>
</body>
</html>
```

上面例子比较简单地说明了如果使用 acl_cpp 中的 HttpServlet/http_mime 等类来实现文件上传的功能，完整的例子请参考：acl_cpp/samples/cig_upload。该例子虽然是一个 CGI 程序，但您依然可以不费吹灰之力将其改变成一个服务器程序，转换方法可参考：《使用 acl_cpp 的 HttpServlet 类及服务器框架编写WEB服务器程序 》。
