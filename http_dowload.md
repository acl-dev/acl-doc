---
title: 使用 acl 库开发一个 HTTP 下载客户端
date: 2010-01-11 18:08:24
tags: http
categories: http开发
---

在 acl 的协议库(lib_protocol) 中有专门针对 HTTP 协议和 ICMP 协议的，本文主要介绍如何使用 lib_protocol 协议库来开发一个简单的 http 客户端。下面首先介绍一下几个本文用到的函数接口。

```c++
/**
 * 创建一个 HTTP_UTIL 请求对象
 * @param url {const char*} 完整的请求 url
 * @param method {const char*} 请求方法，有效的请求方法有：GET, POST, HEAD, CONNECT
 * @return {HTTP_UTIL*}
 */
HTTP_API HTTP_UTIL *http_util_req_new(const char *url, const char *method);

/**
 * 设置 HTTP 代理服务器地址
 * @param http_util {HTTP_UTIL*}
 * @param proxy {const char*} 代理服务器地址，有效格式为: IP:PORT, DOMAIN:PORT,
 *  如: 192.168.0.1:80, 192.168.0.2:8088, www.g.cn:80
 */
HTTP_API void http_util_set_req_proxy(HTTP_UTIL *http_util, const char *proxy);

/**
 * 设置 HTTP 响应体的转储文件，设置后 HTTP 响应体数据便会转储于该文件
 * @param http_util {HTTP_UTIL*}
 * @param filename {const char*} 转储文件名
 * @return {int} 如果返回值 < 0 则表示无法打开该文件, 否则表示打开文件成功
 */
HTTP_API int http_util_set_dump_file(HTTP_UTIL *http_util, const char *filename);

/**
 * 打开远程 HTTP 服务器或代理服务器连接，同时构建 HTTP 请求头数据并且将该数据
 * 发给新建立的网络连接
 * @param http_util {HTTP_UTIL*}
 * @return {int} 0: 成功; -1: 无法打开连接或发送请求头数据失败
 */
HTTP_API int http_util_req_open(HTTP_UTIL *http_util);

/**
 * 发送完请求数据后调用此函数从 HTTP 服务器读取完整的 HTTP 响应头
 * @param http_util {HTTP_UTIL*}
 * @return {int} 0: 成功; -1: 失败
 */
HTTP_API int http_util_get_res_hdr(HTTP_UTIL *http_util);

/**
 * 读完 HTTP 响应头后调用此函数从 HTTP 服务器读取 HTTP 数据体数据，需要连续调用
 * 此函数，直至返回值 <= 0, 如果之前设置了转储文件或转储则在读取数据过程中同时会
 * 拷贝一份数据给转储文件或转储流
 * @param http_util {HTTP_UTIL*}
 * @param buf {char *} 存储 HTTP 响应体的缓冲区
 * @param size {size_t} buf 的空间大小
 * @return {int} <= 0: 表示读结束; > 0: 表示本次读到的数据长度
 */
HTTP_API int http_util_get_res_body(HTTP_UTIL *http_util, char *buf, size_t size);
```

以上仅是 lib_http_util.h 函数接口中的一部分，下面就写一个简单的例子：

```c++
#include "lib_acl.h"
#include "lib_protocol.h"

static void get_url(const char *method, const char *url,
	const char *proxy, const char *dump, int out)
{
	/* 创建 HTTP_UTIL 请求对象 */
	HTTP_UTIL *http = http_util_req_new(url, method);
	int   ret;

	/* 如果设定代理服务器，则连接代理服务器地址，
	 * 否则使用 HTTP 请求头里指定的地址
	 */

	if (proxy && *proxy)
		http_util_set_req_proxy(http, proxy);

	/* 设置转储文件 */
	if (dump && *dump)
		http_util_set_dump_file(http, dump);

	/* 输出 HTTP 请求头内容 */

	http_hdr_print(&http->hdr_req->hdr, "---request hdr---");

	/* 连接远程 http 服务器 */

	if (http_util_req_open(http) < 0) {
		printf("open connection(%s) error\n", http->server_addr);
		http_util_free(http);
		return;
	}

	/* 读取 HTTP 服务器响应头*/

	ret = http_util_get_res_hdr(http);
	if (ret < 0) {
		printf("get reply http header error\n");
		http_util_free(http);
		return;
	}

	/* 输出 HTTP 响应头 */

	http_hdr_print(&http->hdr_res->hdr, "--- reply http header ---");

	/* 如果有数据体则开始读取 HTTP 响应数据体部分 */
	while (1) {
		char  buf[4096];
		
		ret = http_util_get_res_body(http, buf, sizeof(buf) - 1);
		if (ret <= 0)
			break;
		buf[ret] = 0;
		if (out)
			printf("%s", buf);
	}
	http_util_free(http);
}

static void usage(const char *procname)
{
	printf("usage: %s -h[help] -t method -r url -f dump_file -o[output] -X proxy_addr\n"
		"example: %s -t GET -r http://www.sina.com.cn/ -f url_dump.txt\n",
		procname, procname);
}

int main(int argc, char *argv[])
{
	int   ch, out = 0;
	char  url[256], dump[256], proxy[256], method[32];

	acl_init();  /* 初始化 acl 库 */

	ACL_SAFE_STRNCPY(method, "GET", sizeof(method));
	url[0] = 0;
	dump[0] = 0;
	proxy[0] = 0;
	while ((ch = getopt(argc, argv, "hor:t:f:X:")) > 0) {
		switch (ch) {
		case 'h':
			usage(argv[0]);
			return (0);
		case 'o':
			out = 1;
			break;
		case 'r':
			ACL_SAFE_STRNCPY(url, optarg, sizeof(url));
			break;
		case 't':
			ACL_SAFE_STRNCPY(method, optarg, sizeof(method));
			break;
		case 'f':
			ACL_SAFE_STRNCPY(dump, optarg, sizeof(dump));
			break;
		case 'X':
			ACL_SAFE_STRNCPY(proxy, optarg, sizeof(proxy));
			break;
		default:
			break;
		}
	}

	if (url[0] == 0) {
		usage(argv[0]);
		return (0);
	}

	get_url(method, url, proxy, dump, out);
	return (0);
}
```

编译成功后，运行 ./url_get -h 会给出如下提示：
```
usage: ./url_get -h[help] -t method -r url -f dump_file -o[output] -X proxy_addr
example: ./url_get -t GET -r http://www.sina.com.cn/ -f url_dump.txt
```

输入: ./url_get -t GET -r http://www.sina.com -o， 该命令是获取 www.sina.com 页面并输出至标准输出，得到的结果为：
```
HTTP/1.0 301 Moved Permanently
Date: Tue, 12 Jan 2010 01:54:39 GMT
Server: Apache
Location: http://www.sina.com.cn/
Cache-Control: max-age=3600
Expires: Tue, 12 Jan 2010 02:54:39 GMT
Vary: Accept-Encoding
Content-Length: 231
Content-Type: text/html; charset=iso-8859-1
Age: 265
X-Cache: HIT from tj175-135.sina.com.cn
Connection: close
--------------- end -----------------
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>301 Moved Permanently</title>
</head><body>
<h1>Moved Permanently</h1>
<p>The document has moved <a href="http://www.sina.com.cn/">here</a>.</p>
</body></html>
```

如果想把页面转存至文件中，可以输入：./url_get -t GET -r http://www.sina.com -f dump.txt, 这样就会把新浪的首页下载并存储于 dump.txt 文件中。

这个例子非常简单，其实如果查看 http_util.c 源码，会看到这个文件是对 lib_http.h 里一些更为底层 API 的封装。

如果仅是下载一个页面至某个文件中，其实还有更为简单的方法，只需要调用接口：

```c++
/**
 * 将某个 url 的响应体数据转储至某个文件中
 * @param url {const char*} 完整请求 url, 如: http://www.g.cn
 * @param dump {const char*} 转储文件名
 * @param {int} 读到的响应体数据长度, >=0: 表示成功, -1: 表示失败
 */
HTTP_API int http_util_dump_url(const char *url, const char *dump);
```

这一个函数便可以达到与上一个例子相同的效果。