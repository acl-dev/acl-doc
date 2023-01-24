---
title: 使用 acl 库针对 C++ 对象进行序列化及反序列编程
date: 2016-10-16 20:27
categories: 对象序列化
---

在开发网络应用程序时，各个模块之间的数据通信可谓是家常便饭，为了应对这些数据通信时数据交换的要求，程序员发明了各种数据格式：采用二进制数据结构（早期 C 程序员）、采用 XML、采用SOAP（坑人的设计）、采用 URL 编码、采用JSON格式等。客户端与服务端交互时采用这些数据格式进行数据交换时，必然要经历数据编码及数据解码的繁琐过程。早期的二进制数据结构格式对于 C 程序员而是比较简单的，在解码时直接进行结构硬对齐就OK了，但对于其它语言（如 JAVA，PHP）则就麻烦很多，JAVA 语言不仅需要采用字节流一个一个对齐，而且还得要考虑到 C 结构体的数据打包填充方式；SOAP 方式看似满足了各类编程语言数据交换的目的，但数据臃肿是一个很大的问题，我们为了传输几个字节的数据则需要不得不封装大量的 XML 标记。为了使跨语言开发的程序员从麻烦的数据编解码中解脱出来，后来出来很多序列化/反序列化的工具库，比较著名的象 Facebook 的 thrift，Google 的 protobuf。这些工具库功能非常强大，但有一个问题是，这些工具库所要求的预先定义的 schema 的亲民性不够好，增加了额外的大量学习成本。

最近由 niukey@qq.com 为 acl 库新增了 C++ 对象序列化与反序列化库，大大提高了程序员的编程效率及代码准确率。acl 序列化库实现了 C++ struct 对象与 JSON 对象之间互转功能，使用方式非常简单。

## 一、acl 序列化库的功能特点：

 - 可以直接将 C++ struct 对象转换为 Json 对象，同时还可以将 Json 对象反序列化为 C++ struct 对象；
- 支持 C++ struct 对象中包含构造函数及析构函数；
- C++ struct 对象中的成员支持常见基本类型（short, int, long, long long, char*）、标准 C++ 类对象；
- C++ struct 对象中的成员支持指针类型；
- C++ struct 对象中的成员支持常见 C++ 容器类型：std::vector、std::list、std::map，且支持容器对象的内部嵌套；
- C++ struct 对象中的成员为基本数据类型（如：short, int, long, long long）和指针类型时，支持直接在 struct 中针对这些成员进行初始化（C++11）；
- 支持在 C++ struct 中添加注释（// 或 /**/）；
- 支持 C++ struct 对象的多继承；
- 支持在 C++ struct 对象中的多级包含，建议使用包含方式代替继承方式；
- 支持 C++ struct 成员增加注释项：//Gson@optional 表示该成员变量为可选项，//Gson@required 表示该成员为必须项，默认为必须的。

## 二、使用 acl 序列化库的限制：

- struct 中的成员类型不能为 char 类型，主要是因为 Json 对象中不存在 char 类型；
- struct 中的成员前不得有 const 常量限定词，主要是在反序列化时需要对这些成员进行赋值；
- struct 中不支持 C++ 关键词：public、private、protected；
- struct 中的成员变量名不得为：$json、$node、$obj，因为这些关键词被 acl 序列化库生成的代码占用；
-存在包含关系的 struct 类型定义必须在同一个文件中；
- 不支持纯 C 语言的数组功能，比如，针对 int a[10]，因无法确切知道数组 a 中真实存储的数量而无法进行序列化和反序列化，因此推荐使用 std::vector<int> 代替它。

## 三、使用 acl 序列化库的过程：

- 首先编译 acl 库：在 acl 目录前运行：make build_one，会生成 acl 的三个基础库（lib_acl/lib/lib_acl.a, lib_protocol/lib/lib_protocol.a, lib_acl_cpp/liblib_acl_cpp.a），同时还会在 acl 根目录下生成这三个库的合集（lib_acl.a 和 lib_acl.so）；
- 编译 acl 序列化工具：进入 app/gson 目录，运行 make，生成 gson 工具，该工具将被用于根据用户自定义 struct 生成序列化 C++ 代码；
- 定义自己的 struct 结构体（可以定义多个）并保存在文件中（文件后缀名为 .stub）；
- 使用 gson 工具根据用户的 .stub 文件生成序列化所需的 C++ 代码：./gson -d path，其中 path 为保含 .stub 文件的目录，将会生成三个文件 gson.cpp、 gson.h 和 由 .stub 转换的 .h 头文件；
- 将 gson.h 和由 .stub 文件转换的 .h 头文件包含在自己的代码中；
- 针对用户自定义的每一个 struct，在 gson.h 头文件中均会提供 struct 对象的序列化和反序列化的方法，假设用户自定义了一个结构体类型：struct user，则在由工具 gson 生成的 gson.h 文件中包含如下五个方法：
  ```
  acl::string gson(const user &$obj);  

    将用户填充好的 user 对象 $obj 转换为 Json 字符串；
  acl::json_node& gson(acl::json &$json, const user &$obj);

    将用户填充好的 user 对象 $obj 转化为 acl::json 对象 $json 中的一个 acl::json_node 节点对象并将之返回，这样用户可以将这个返回的 acl::json_node& 对象引用添加于 $json 对象中；

  acl::json_node& gson(acl::json &$json, const user *$obj);

     功能与 2）中的方法相同，只是 $obj 参数为对象指针；

  std::pair<bool,std::string> gson(acl::json_node &$node, user &$obj);

     将一个 acl::json_node Json 节点对象转化为用户自定义的 struct user 对象，如果 $node 为该 Json 对象中的根节点，则该根节点可由 $json.get_root() 获得，$obj 存储转换后的结果；

  std::pair<bool,std::string> gson(acl::json_node &$node, user *$obj);

     功能与 5）中的方法相同，只是 $obj 参数为对象指针。
     ```

## 四、举例：

假设自定义结构对象如下：

```c++
// struct.stub
#pragma once
#include <string>

struct user
{
        std::string name;
        std::string domain;
        int age;
        bool male;
};

      应用操作 user 对象的过程如下：

// main.cpp
#include "stdafx.h"
#include <list>
#include <vector>
#include <map>
#include <stdio.h>
#include <iostream>
#include <time.h>
#include "struct.h"  // 由 gson 工具根据 struct.stub 转换而成
#include "gson.h"   // 由 gson 工具根据 struct.stub 生成

// 序列化过程
static void serialize(void)
{
	user u;

	u.name = "zsxxsz";
	u.domain = "263.net";
	u.age = 11;
	u.male = true;

	acl::json json;

	// 将 user 对象转换为 json 对象
	acl::json_node& node = acl::gson(json, u);

	printf("serialize:\r\n");
	printf("json: %s\r\n", node.to_string().c_str());
	printf("\r\n");
}

// 反序列化过程
static void deserialize(void)
{
	const char *s = "{\"name\": \"zsxxsz\", \"domain\": \"263.net\", \"age\": 11, \"male\": true}";
	printf("deserialize:\r\n");

	acl::json json;
	json.update(s);
	user u;

	// 将 json 对象转换为 user 对象
	std::pair<bool, std::string> ret = acl::gson(json.get_root(), u);

	// 如果转换失败，则打印转换失败原因
	if (ret.first == false)
		printf("error: %s\r\n", ret.second.c_str());
	else
		printf("name: %s, domain: %s, age: %d, male: %s\r\n",
			u.name.c_str(), u.domain.c_str(), u.age,
			u.male ? "yes" : "no");
}

int main(void)
{
	serialize();
	deserialize();
	return 0;
}
```

## 五、参考：

github：https://github.com/acl-dev/acl
gitee：https://gitee.com/acl-dev/acl
 

 