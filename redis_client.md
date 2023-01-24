---
title: 使用 acl 库编写高效的 C++ redis 客户端应用
date: 2015-02-10 01:03
categories: redis使用
---

## 一、概述
（可以直接略过此段）redis 最近做为 nosql 数据服务应用越来越广泛，其相对于 memcached 的最大优点是提供了更加丰富的数据结构，所以应用场景就更为广泛。redis 的出现可谓是广大网络应用开发者的福音，同时有大量的开源人员贡献了客户端代码，象针对 java 语言的 jedis，php 语言的 phpredis/predis 等，这些语言的 redis 库既丰富又好用，而对 C/C++ 程序员似乎就没那么幸运了，官方提供了 C 版的 hiredis 作为客户端库，很多爱好者都是基于 hiredis 进行二次封装和开发形成了 C++ 客户端库，但这些库（包括官方的 hiredis）大都使用麻烦，给使用者造成了许多出错的机会。一直想开发一个更易用的接口型的 C++ 版 redis 客户端库（注：官方提供的库基本属于协议型的，这意味着使用者需要花费很多精力去填充各个协议字段同时还得要分析服务器可能返回的不同的结果类型），但每当看到 redis 那 150 多个客户端命令时便心生退缩，因为要给每个命令提供一个方便易用的 C++ 函数接口，则意味着非常巨大的开发工作量。

在后来的多次项目开发中被官方的 hiredis 库屡次摧残后，终于忍受不了，决定重新开发一套全新的 redis 客户端 API，该库不仅要实现这 150 多个客户端命令，同时需要提供方便灵活的连接池及连接池集群管理功能（幸运的是在 acl 库中已经具备了通用的网络连接池及连接池集群管理模块），另外，根据之前的实践，有可能提供的函数接口要远大于这 150 多个，原因是针对同一个命令可能会因为不同的参数类型场景提供多个函数接口（最终的结果是提供了3，4百个函数 API，7000+行源码，2000+行头文件）；在仔细研究了 redis 的通信协议后便着手开始进行设计开发了（redis 的协议设计还是非常简单实用的，即能支持二进制，同时又便于手工调试）。在开发过程中大量参考了 http://redisdoc.com 网站上的中文在线翻译版（非常感谢 黄键宏 同学的辛勤工作）。

## 二、acl redis 库分类
根据 redis 的数据结构类型，分成 12 个大类，每个大类提供不同的函数接口，这 12 个 C++ 类展示如下：

- 1、redis_key：redis 所有数据类型的统一键操作类；因为 redis 的数据结构类型都是基本的 KEY-VALUE 类型，其中 VALUE 分为不同的数据结构类型；
- 2、redis_connectioin：与 redis-server 连接相关的类；
- 3、redis_server：与 redis-server 服务管理相关的类；
- 4、redis_string：redis 中用来表示字符串的数据类型；
- 5、redis_hash：redis 中用来表示哈希表的数据类型；每一个数据对象由 “KEY-域值对集合” 组成，即一个 KEY 对应多个“域值对”，每个“域值对”由一个字段名与字段值组成；
- 6、redis_list：redis 中用来表示列表的数据类型；
- 7、redis_set：redis 中用来表示集合的数据类型；
- 8、redis_zset：redis 中用来表示有序集合的数据类型；
- 9、redis_pubsub：redis 中用来表示“发布-订阅”的数据类型；
- 10、redis_hyperloglog：redis 中用来表示 hyperloglog 基数估值算法的数据类型；
- 11、redis_script：redis 中用来与 lua 脚本进行转换交互的数据类型；
- 12、redis_transaction：redis 中用以事务方式执行多条 redis 命令的数据类型（注：该事务处理方式与数据库的事务有很大不同，redis 中的事务处理过程没有数据库中的事务回滚机制，仅能保证其中的多条命令都被执行或都不被执行）；

除了以上对应于官方 redis 命令的 12 个类别外，在 acl 库中还提供了另外几个类：

- 13、redis_command：以上 12 个类的基类；
- 14、redis_client：redis 客户端网络连接类；
- 15、redis_result：redis 命令结果类；
- 16、redis_pool：针对以上所有命令支持连接池方式；
- 17、redis_manager：针对以上所有命令允许与多个 redis-server 服务建立连接池集群（即与每个 redis-server 建立一个连接池）；
- 18、redis_cluster：支持 redis3.0 集群模式的类。

## 三、acl redis 使用举例
### 3.1、下面是一个使用 acl 框架中 redis 客户端库的简单例子：

```c++
/**
 * @param conn {acl::redis_client&} redis 连接对象
 * @return {bool} 操作过程是否成功
 */
bool test_redis_string(acl::redis_client& conn, const char* key)
{
	// 创建 redis string 类型的命令操作类对象，同时将连接类对象与操作类
	// 对象进行绑定
	acl::redis_string string_operation(&conn);
	const char* value = "test_value";

	// 添加 K-V 值至 redis-server 中
	if (string_operation.set(key, value) == false)
	{
		const acl::redis_result* res = string_operation.get_result();
		printf("set key: %s error: %s\r\n",
			key, res ? res->get_error() : "unknown error");
		return false;
	}
	printf("set key: %s ok!\r\n", key);

	// 需要重置连接对象的状态
	string_operation.clear();

	// 从 redis-server 中取得对应 key 的值
	acl::string buf;
	if (string_operation.get(key, buf) == false)
	{
		const acl::redis_result* res = string_operation.get_result();
		printf("get key: %s error: %s\r\n",
			key, res ? res->get_error() : "unknown error");
		return false;
	}
	printf("get key: %s ok, value: %s\r\n", key, buf.c_str());

	// 探测给定 key 是否存在于 redis-server 中，需要创建 redis 的 key
	// 类对象，同时将 redis 连接对象与之绑定
	acl::redis_key key_operation;
	key_operation.set_client(conn);  // 将连接对象与操作对象进行绑定
	if (key_operation.exists(key) == false)
	{
		if (conn.eof())
		{
			printf("disconnected from redis-server\r\n");
			return false;
		}

		printf("key: %s not exists\r\n", key);
	}
	else
		printf("key: %s exists\r\n", key);

	// 删除指定 key 的字符串类对象
	if (key_operation.del(key, NULL) < 0)
	{
		printf("del key: %s error\r\n", key);
		return false;
	}
	else
		printf("del key: %s ok\r\n", key);

	return true;
}

/**
 * @param redis_addr {const char*} redis-server 服务器地址，
 *  格式为：ip:port，如：127.0.0.1:6379
 * @param conn_timeout {int} 连接 redis-server 的超时时间(秒)
 * @param rw_timeout {int} 与 redis-server 进行通信的 IO 超时时间(秒)
 */
bool test_redis(const char* redis_addr, int conn_timeout, int rw_timeout)
{
	// 创建 redis 客户端网络连接类对象
	acl::redis_client conn(redis_addr, conn_timeout, rw_timeout);
	const char* key = "test_key";
	return test_redis_string(conn, key);
}
```

上面的简单例子的操作过程是：在 redis-server 中添加字符串类型数据 --> 从 redis-server 中获取指定的字符串数据 --> 判断指定指定 key 的对象在 redis-server 上是否存在 ---> 从 redis-server 中删除指定 key 的数据对象（即该例中的字符串对象）。通过以上简单示例，使用者需要注意以下几点：

- acl 中的 redis 库的设计中 redis 连接类对象与命令操作类对象是分离的，12 个 redis 命令操作类对应 acl  redis 库中相应的 12 个命令操作类；
- 在使用 redis 命令操作类时需要先将 redis 连接类对象与命令操作类对象进行绑定（以便于操作类内部可以利连接类中的网络连接、协议组包以及协议解析等方法）；
- 在重复使用一个 redis 命令类对象时，需要首先重置该命令类对象的状态（即调用：clear()），这样主要是为了释放上一次命令操作过程的中间内存资源；
- 一个 redis 连接类对象可以被多个命令类操作类对象使用（使用前需先绑定一次）；
- 将 redis 连接对象与命令操作对象绑定有两种方式：可以在构造函数中传入非空 redis 连接对象，或调用操作对象的 set_client 方法进行绑定。

### 3.2、对上面的例子稍加修改，使之能够支持连接池方式，示例代码如下：
```c++
/**
 * @param conn {acl::redis_client&} redis 连接对象
 * @return {bool} 操作过程是否成功
 */
bool test_redis_string(acl::redis_client& conn, const char* key)
{
	...... // 代码与上述代码相同，省略

	return true;
}

// 子线程处理类
class test_thread : public acl::thread
{
public:
	test_thread(acl::redis_pool& pool) : pool_(pool) {}

	~test_thread() {}

protected:
	// 基类（acl::thread）纯虚函数
	virtual void* run()
	{
		acl::string key;
		// 给每个线程一个自己的 key，以便以测试，其中 thread_id()
		// 函数是基类 acl::thread 的方法，用来获取线程唯一 ID 号
		key.format("test_key: %lu", thread_id());

		acl::redis_client* conn;

		for (int i = 0; i < 1000; i++)
		{
			// 从 redis 客户端连接池中获取一个 redis 连接对象
			conn = (acl::redis_client*) pool_.peek();
			if (conn == NULL)
			{
				printf("peek redis connection error\r\n");
				break;
			}

			// 进行 redis 客户端命令操作过程
			if (test_redis_string(*conn) == false)
			{
				printf("redis operation error\r\n");
				break;
			}

			// 回收连接对象
			pool_.put(conn, true);
		}

		return NULL;
	}

private:
	acl::redis_pool& pool_;
};

void test_redis_pool(const char* redis_addr, int max_threads,
	int conn_timeout, int rw_timeout)
{
	// 创建 redis 连接池对象
	acl::redis_client_pool pool(redis_addr, max_threads);
	// 设置连接 redis 的超时时间及 IO 超时时间，单位都是秒
	pool.set_timeout(conn_timeout, rw_timeout);

	// 创建一组子线程
	std::vector<test_thread*> threads;
	for (int i = 0; i < max_threads; i++)
	{
		test_thread* thread = new test_thread(pool);
		threads.push_back(thread);
		thread->set_detachable(false);
		thread->start();
	}

	// 等待所有子线程正常退出
	std::vector<test_thread*>::iterator it = threads.begin();
	for (; it != threads.end(); ++it)
	{
		(*it)->wait();
		delete (*it);
	}
}
```

除了创建线程及 redis 连接池外，上面的例子与示例 1） 的代码与功能无异。

### 3.3、下面对上面的示例2）稍作修改，使之可以支持 redis 集群连接池的方式，示例代码如下：
```c++
/**
 * @param conn {acl::redis_client&} redis 连接对象
 * @return {bool} 操作过程是否成功
 */
bool test_redis_string(acl::redis_client& conn, const char* key)
{
	......  // 与上面示例代码相同，略去
	return true;
}

// 子线程处理类
class test_thread : public acl::thread
{
public:
	test_thread(acl::redis_cluster& cluster) : cluster_(cluster) {}

	~test_thread() {}

protected:
	// 基类（acl::thread）纯虚函数
	virtual void* run()
	{
		acl::string key;
		acl::redis_client_pool* pool;
		acl::redis_client* conn;

		for (int i = 0; i < 1000; i++)
		{
			// 从连接池集群管理器中获得一个 redis-server 的连接池对象
			pool = (acl::redis_client_pool*) cluster_.peek();
			if (pool == NULL)
			{
				printf("peek connection pool failed\r\n");
				break;
			}

			// 从 redis 客户端连接池中获取一个 redis 连接对象
			conn = (acl::redis_client*) pool_.peek();
			if (conn == NULL)
			{
				printf("peek redis connection error\r\n");
				break;
			}

			// 给每个线程一个自己的 key，以便以测试，其中 thread_id()
			// 函数是基类 acl::thread 的方法，用来获取线程唯一 ID 号
			key.format("test_key: %lu_%d", thread_id(), i);
			// 进行 redis 客户端命令操作过程
			if (test_redis_string(*conn, key.c_str()) == false)
			{
				printf("redis operation error\r\n");
				break;
			}

			// 回收连接对象至连接池中
			pool_.put(conn, true);
		}

		return NULL;
	}

private:
	acl::redis_cluster& cluster_;
};

void test_redis_manager(const char* redis_addr, int max_threads,
	int conn_timeout, int rw_timeout)
{
	// 创建 redis 集群连接池对象
	acl::redis_client_cluster cluster;

	// 添加多个 redis-server 的服务器实例地址
	cluster.set("127.0.0.1:6379", max_threads, conn_timeout, rw_timeout);
	cluster.set("127.0.0.1:6380", max_threads, conn_timeout, rw_timeout);
	cluster.set("127.0.0.1:6381", max_threads, conn_timeout, rw_timeout);

	// 创建一组子线程
	std::vector<test_thread*> threads;
	for (int i = 0; i < max_threads; i++)
	{
		test_thread* thread = new test_thread(cluster);
		threads.push_back(thread);
		thread->set_detachable(false);
		thread->start();
	}

	// 等待所有子线程正常退出
	std::vector<test_thread*>::iterator it = threads.begin();
	for (; it != threads.end(); ++it)
	{
		(*it)->wait();
		delete (*it);
	}
}
```

该示例只修改了几处代码便支持了集群 redis 连接池方式，其处理过程是：创建集群连接池对象（可以添加多个 redis-server 服务地址） --> 从集群连接池对象中取得一个连接池对象 ---> 从该连接池对象中取得一个连接 ---> 该连接对象与 redis 操作类对象绑定后进行操作。

上述示例的集群模式并非是 redis3.0 的集群模式，这种集群中的 redis-server 之间是不互联的， 集群的建立是由客户端来维护的，由客户决定数据存储在哪个 redis-server 实例上；而 redis3.0 的集群方式则与之大不同，在 redis3.0 中，redis-server 之间是互联互通的，而且支持数据的冗余备份，数据的存储位置是由服务端决定的，下面的例子是支持 redis3.0 集群模式的客户端例子：

### 3.4、支持 redis3.0 集群模式的示例代码如下：
```c++
// 统一的键值前缀
static acl::string __keypre("test_key_cluster");

// 测试 redis 字符串添加功能
static bool test_redis_string(acl::redis_string& cmd, int i)
{
	acl::string key;
	key.format("%s_%d", __keypre.c_str(), i);

	acl::string value;
	value.format("value_%s", key.c_str());
	
	bool ret = cmd.set(key.c_str(), value.c_str());
	return ret;
	if (i < 10)
		printf("set key: %s, value: %s %s\r\n", key.c_str(),
			value.c_str(), ret ? "ok" : "error");
	return ret;
}

// 测试 redis 键是否存在功能
static bool test_redis_exists(acl::redis_key& cmd, int i)
{
	acl::string key;

	key.format("%s_%d", __keypre.c_str(), i);

	if (cmd.exists(key.c_str()) == false)
	{
		if (i < 10)
			printf("no exists key: %s\r\n", key.c_str());
	}
	else
	{
		if (i < 10)
			printf("exists key: %s\r\n", key.c_str());
	}
	return true;
}

// 子线程处理类
class test_thread : public acl::thread
{
public:
	test_thread(acl::redis_cluster& cluster, int max_conns)
	: cluster_(cluster), max_conns_(max_conns) {}

	~test_thread() {}

protected:
	// 基类（acl::thread）纯虚函数
	virtual void* run()
	{
		acl::redis_string cmd_string;
		acl::redis_key  cmd_key;
		
		// 设置 redis 客户端命令的集群操作模式
		cmd_key.set_cluster(&cluster_, max_conns_);
		cmd_string.set_cluster(&cluster_, max_conns_);
		for (int i = 0; i < 1000; i++)
		{
			// 进行 redis 客户端命令操作过程
			if (test_redis_string(cmd_string, i) == false)
				break;
	
			if (test_redis_exists(cmd_key, i) == false)
				break;

			// 重置客户端命令状态

			cmd_string.clear();
			cmd_key.clear();
		}

		return NULL;
	}

private:
	acl::redis_cluster& cluster_;
	int max_conns_;
};

void test_redis_cluster(int max_threads int conn_timeout, int rw_timeout)
{
	// 创建 redis 集群连接池对象
	acl::redis_client_cluster cluster;

	// 添加一个或多个 redis-server 的服务器实例地址，不必一次加载所有
	// 的 redis-server 服务器地址，redis_cluster 及相关类具有自动发现
	// 及动态添加 redis-server 服务实例的功能

	cluster.set("127.0.0.1:6379", max_threads, conn_timeout, rw_timeout);
	// cluster.set("127.0.0.1:6380", max_threads, conn_timeout, rw_timeout);
	// cluster.set("127.0.0.1:6381", max_threads, conn_timeout, rw_timeout);

	// 创建一组子线程
	std::vector<test_thread*> threads;
	for (int i = 0; i < max_threads; i++)
	{
		test_thread* thread = new test_thread(cluster, max_threads);
		threads.push_back(thread);
		thread->set_detachable(false);
		thread->start();
	}

	// 等待所有子线程正常退出
	std::vector<test_thread*>::iterator it = threads.begin();
	for (; it != threads.end(); ++it)
	{
		(*it)->wait();
		delete (*it);
	}
}
```

从上面例子可以看出，使用 acl redis 客户端库操作 redis3.0 集群时，只需要将集群的句柄注入到每个 redis 客户端命令即（如上面黄色部分所示）；至于如何与 redis 集群交互将由 redis_cluster 及 redis 客户端命令类的基类 redis_command 进行处理；此外，还需要注意示例 4）与示例 3）所支持的集群方式的不同点如下：

- 示例3 的集群模式下实际上是由客户端根据所连接的所有 redis 服务器的集合来决定数据存储结点，而实际上 redis 服务器之间并不互联；而示例 4 则是真正意义的 redis 服务端的集群模式，所有 redis 服务结点之间是互联互通的（可以配置数据结点存储的冗余数量）；
- 示例3 的客户端必须在开始初始化时添加所有的 redis 服务结点，以便于采用轮循或者哈希访问模式；而示例4 在初始化时只需添加至少一个集群中的服务结点即可，随着访问次数的增加，会根据需要动态添加 redis 服务结点（ redis3.0 采用的重定向机制，即当访问某个 redis 结点时，若 key 值不存在于该结点上，则其会返回给客户端一个重定向指令，告诉客户端存储该 key 的 redis 服务结点，因此，根据此特性，acl redis 集群会根据重定向信息动态添加 redis 集群中的服务结点）；
- 此外，示例4 是兼容示例3 的。

## 四、小结
以上介绍了 acl 框架中新增加的 redis 库的使用方法及处理过程，该库将复杂的协议及网络处理过程隐藏在实现内部，使用户使用起来感觉象是在调用本的函数。在示例 2）、3） 中提到了 acl 线程的使用，有关 acl 库中更为详细地使用线程的文章参见：《使用 acl_cpp 库编写多线程程序》。

github：https://github.com/acl-dev/acl
gitee：https://gitee.com/acl-dev/acl

 