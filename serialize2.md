---
title: c++对象序列化编程实例 编辑
date: 2016-11-14 22:39
categories: 对象序列化
---

在《使用 acl 库针对 C++ 对象进行序列化及反序列编程》中介绍了 acl 库中针对 C/C++ 的 struct 对象进行序列化和反序列化的功能，并且给出了一个简单的例子。本文将介绍一些较为复杂的例子。

## 一、示例一：支持多继承的例子

先定义 struct.stub 文件：

```c++
#pragma once
#include <string>

struct student
{
	std::string shcool;
	std::string class_name;
};

struct province
{
	std::string province_name;
	std::string position;
};

struct user : student, province
{
	std::string name;
	int  age;
	bool male;
};
```

上面的定义中，user 继承于 student 和 province。

然后使用 gson 工具（运行：./gson -d ）根据此 struct.stub 生成目标源文件和头文件：gson.cpp, gson.h, struct.h。

下面就可以编写业务逻辑代码：

```c++
#include "stdafx.h"
#include <list>
#include <vector>
#include <map>
#include <stdio.h>
#include <iostream>
#include <time.h>
#include "struct.h"  // 由 gson 工具根据 struct.stub 转换而成
#include "gson.h"    // 由 gson 工具根据 struct.stub 生成

// 序列化过程
static void serialize(void)
{
	user u;

	u.name = "zsx";
	u.age = 11;
	u.male = true;

	u.province_name = "省";
	u.position = "位置";

	u.shcool = "大学";
	u.class_name = "班级";

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
	const char *s = "{\"shcool\": \"大学\", \"class_name\": \"班级\", \"province_name\": \"省\", \"position\": \"位置\", \"name\": \"zsx\", \"age\": 11, \"male\": true}";
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
	{
		printf("name: %s, age: %d, male: %s\r\n",
			u.name.c_str(), u.age, u.male ? "yes" : "no");
		printf("province_name: %s, position: %s\r\n",
			u.province_name.c_str(), u.position.c_str());
		printf("shcool: %s, class_name: %s\r\n",
			u.shcool.c_str(), u.class_name.c_str());
	}
}

int main(void)
{
	serialize();
	deserialize();
	return 0;
}
```

编译并运行该例子，结果如下：

```
serialize:
json: {"shcool": "大学", "class_name": "班级", "province_name": "省", "position": "位置", "name": "zsx", "age": 11, "male": true}

deserialize:
name: zsx, age: 11, male: yes
province_name: 省, position: 位置
shcool: 大学, class_name: 班级
```

## 二、示例二：支持 C++11 的例子

 定义 struct.stub 文件如下：

```c++
#pragma once

struct user
{
	// 带参数的构造函数
	user(const char* user_name, const char* user_domain,
		int user_age, bool user_male)
	: username(user_name)
	, domain(user_domain)
	, age(user_age)
	, male(user_male)
	{}

	user() {}
	~user() {}

	acl::string username;
	acl::string domain;
	int age = 100;
	bool male = true;
};

struct message
{
	int type;
	acl::string cmd;
	std::list<user> user_list;
	std::list<user> user_vector;
	std::map<acl::string, user> user_map;

	std::list<user*> *user_list_ptr = nullptr;
	std::vector<user*> *user_vector_ptr = nullptr;
	std::map<acl::string, user*> *user_map_ptr = nullptr;

	int n = 100;				// c++11 允许初始化成员变量
	long n1 = 1000;
	long long int n2 = 1000;
	short n3 = 100;
	//Gson@optional
	user* u = nullptr;

	message() {}

	~message()
	{
		if (user_list_ptr)
		{
			for (auto it : *user_list_ptr)
				delete it;
			delete user_list_ptr;
		}
		if (user_vector_ptr)
		{
			for (auto it : *user_vector_ptr)
				delete it;
			delete user_vector_ptr;
		}
		if (user_map_ptr)
		{
			for (auto it : *user_map_ptr)
				delete it.second;
			delete user_map_ptr;
		}
	}
};
```

用 gson 工具将上述 stub 文件生成同样的三个文件：gson.cpp, gson.h, struct.h，然后编写业务处理代码如下：

```c++
#include "stdafx.h"
#include <list>
#include <vector>
#include <map>
#include <stdio.h>
#include <iostream>
#include <time.h>
#include "struct.h"
#include "gson.h"

static void print_msg(message& msg)
{
	printf("=======================================================\r\n");
	printf("type: %d\r\n", msg.type);
	printf("cmd: %s\r\n", msg.cmd.c_str());

	printf("-------------------- user list ------------------------\r\n");
	size_t i = 0;
	for (auto cit : msg.user_list)
	{
		printf(">>username: %s, domain: %s, age: %d, male: %s\r\n",
			cit.username.c_str(), cit.domain.c_str(),
			cit.age, cit.male ? "true" : "false");
		if (++i >= 10)
			break;
	}

	printf("-------------------- user vector ----------------------\r\n");
	i = 0;
	for (auto cit : msg.user_vector)
	{
		printf(">>username: %s, domain: %s, age: %d, male: %s\r\n",
			cit.username.c_str(), cit.domain.c_str(),
			cit.age, cit.male ? "true" : "false");
		if (++i >= 10)
			break;
	}

	printf("------------------- user map --------------------------\r\n");
	i = 0;
	for (auto cit : msg.user_map)
	{
		printf(">>key: %s, username: %s, domain: %s, age: %d, male: %s\r\n",
			cit.first.c_str(),
			cit.second.username.c_str(),
			cit.second.domain.c_str(),
			cit.second.age,
			cit.second.male ? "true" : "false");
		if (++i >= 10)
			break;
	}
	printf("-------------------- user list ptr --------------------\r\n");
	i = 0;
	for (auto cit : *msg.user_list_ptr)
	{
		printf(">>username: %s, domain: %s, age: %d, male: %s\r\n",
			cit->username.c_str(), cit->domain.c_str(),
			cit->age, cit->male ? "true" : "false");
		if (++i >= 10)
			break;
	}

	printf("-------------------- user vector ptr ------------------\r\n");
	i = 0;
	for (auto cit : *msg.user_vector_ptr)
	{
		printf(">>username: %s, domain: %s, age: %d, male: %s\r\n",
			cit->username.c_str(), cit->domain.c_str(),
			cit->age, cit->male ? "true" : "false");
		if (++i >= 10)
			break;
	}

	printf("------------------- user map ptr ----------------------\r\n");
	i = 0;
	for (auto cit : *msg.user_map_ptr)
	{
		printf(">>key: %s, username: %s, domain: %s, age: %d, male: %s\r\n",
			cit.first.c_str(),
			cit.second->username.c_str(),
			cit.second->domain.c_str(),
			cit.second->age,
			cit.second->male ? "true" : "false");
		if (++i >= 10)
			break;
	}

	printf("-------------------------------------------------------\r\n");
}

static void test(void)
{
	//////////////////////////////////////////////////////////////////
	// 序列化过程

	message msg;
	msg.user_list_ptr   = new std::list<user*>;
	msg.user_vector_ptr = new std::vector<user*>;
	msg.user_map_ptr    = new std::map<acl::string, user*>;

	msg.type = 1;
	msg.cmd  = "add";

	user u = {"zsx1", "263.net", 11, true};
	msg.user_list.push_back(u);
	msg.user_list.emplace_back(u);
	msg.user_list.emplace_back("zsx1", "263.net", 13, false);

	u = {"zsx2", "263.net", 11, true};
	msg.user_vector.push_back(u);
	msg.user_vector.emplace_back(u);
	msg.user_vector.emplace_back("zsx2", "263.net4", 14, true);

	u = {"zsx31", "263.net", 11, true};
	msg.user_map[u.username] = u;
	msg.user_map["zsx32"] = {"zsx32", "263.net", 11, true };

	msg.user_list_ptr->push_back(new user("zsx4", "263.net1", 11, true));
	msg.user_list_ptr->push_back(new user("zsx4", "263.net2", 12, true));

	msg.user_vector_ptr->push_back(new user("zsx5", "263.net1", 11, true));
	msg.user_vector_ptr->push_back(new user("zsx5", "263.net2", 12, true));

	(*msg.user_map_ptr)["zsx61"] = new user("zsx61:", "263.net1", 11, true);
	(*msg.user_map_ptr)["zsx62"] = new user("zsx62", "263.net2", 12, true);

	acl::json json;

	// 序列化
	acl::json_node& node = acl::gson(json, msg);
	printf("serialize: %s\r\n", node.to_string().c_str());

	/////////////////////////////////////////////////////////////////////
	// 反序列化过程

	message msg1;
	acl::json json1;
	json1.update(node.to_string());

	printf("json1 to_string: %s\r\n", json1.to_string().c_str());

	// 反序列化
	std::pair<bool, std::string> ret = acl::gson(json1.get_root(), msg1);
	if (ret.first == false)
		printf("error: %s\r\n", ret.second.c_str());
	else
		print_msg(msg);
}

int main(void)
{
	test();
	return 0;
}

       上述例子主要体现了 gson 的两个特点：1、允许有构造函数，2、支持 C++11 中的结构成员初始化。
```

## 三、参考

github：https://github.com/acl-dev/acl
gitee：https://gitee.com/acl-dev/acl