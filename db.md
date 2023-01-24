---
title: 使用 acl 库编写数据库应用程序
date: 2014/09/03 12:11
categories: 数据库编程
---

acl 的 C++ 版本库（lib_acl_cpp.a）的 db 模块主要与数据库编程相关，通过这些模块库，开发者可以快速地写出支持数据库连接池的数据库应用程序，目前该 db 模块支持 mysql、sqlite 数据库。本文将以 mysql 应用为例讲述如何使用这些 API 接口编程数据库应用。

在 lib_acl_cpp/include/acl_cpp/db 目录下，可以看到主要分三个部分：数据库操作句柄类（db_handle，db_mysql，db_sqlite）、数据库连接池类（db_pool，mysql_pool，sqlite_pool）及数据库服务类（db_service，db_service_mysql，db_service_sqlite，这些类主要用在阻塞非阻塞结合的应用中，如：MFC界面过程与数据库过程的结合，非阻塞 IO 过程与数据库过程结合）。

## 一、数据库操作句柄

下图显示了数据库句柄的类继承关系：db_handle 为基础类，db_mysql/db_sqlite 类均继承于 db_handle 类。

![db_handle基类](/img/db.png)

在 db_mysql.hpp/db_sqlite.hpp 两个头文件中可以看出，这两个子类仅是实现了基础类 db_handle 的一些虚函数而已，大量关于数据的操作函数都集中于 db_handle.hpp 头文件中，下图为 db_handle 类的功能协作图（其中的 db_rows/db_row 两个类为数据库查询结果类）：
![db继承关系](/img/db2.png)

下面给出了一个简单的数据库查询示例：

```c++
////////////////////////////////////////////////////////////////////////////////
/**
 * 从数据库中查询表数据
 * @param db {acl::db_handle&} acl 中的数据库连接句柄引用
 */
static void tbl_select(acl::db_handle& db)
{
	// 创建 sql 查询语句
	const char* sql = "select * from group_tbl where"
		" group_name='test_name' and uvip_tbl='test'";
	// 查询数据库
	if (db.sql_select(sql) == false)
	{
		printf("select sql: %s error\r\n", sql);
		return;
	}
	printf("\r\n---------------------------------------------------\r\n");
	// 列出查询结果方法一：从数据库句柄中获得查询结果集合
	const acl::db_rows* result = db.get_result();
	if (result)
	{
		// 遍历查询结果集
		const std::vector<acl::db_row*>& rows = result->get_rows();
		for (size_t i = 0; i < rows.size(); i++)
		{
			const acl::db_row* row = rows[i];
			// 打印一行结果中的所有结果
			for (size_t j = 0; j < row->length(); j++)
				printf("%s, ", (*row)[j]);
			printf("\r\n");
		}
	}
	// 列出查询结果方法二：根据数组下标遍历数据库句柄的查询结果集
	for (size_t i = 0; i < db.length(); i++)
	{
		const acl::db_row* row = db[i];

		// 取出该行记录中某个字段的值
		const char* ptr = (*row)["group_name"];
		if (ptr == NULL)
		{
			printf("error, no group name\r\n");
			continue;
		}
		printf("group_name=%s: ", ptr);
		for (size_t j = 0; j < row->length(); j++)
			printf("%s, ", (*row)[j]);
		printf("\r\n");
	}
	// 列出查询结果方法三：直接从数据库句柄中获得结果数组
	const std::vector<acl::db_row*>* rows = db.get_rows();
	if (rows)
	{
		std::vector<acl::db_row*>::const_iterator cit = rows->begin();
		for (; cit != rows->end(); cit++)
		{
			const acl::db_row* row = *cit;
			for (size_t j = 0; j < row->length(); j++)
				printf("%s, ", (*row)[j]);
			printf("\r\n");
		}
		
	}
	// 必须释放查询结果
	db.free_result();
}
////////////////////////////////////////////////////////////////////////////////
/**
 * 打开 mysql 数据库连接句柄
 * @return {acl::db_handle*} 返回值为 NULL 表示连接数据库失败
 */
static acl::db_handle* open_mysql(void)
{
	const char* dbaddr = "127.0.0.1:3306";
	const char* dbname = "acl_test_db";
	const char* dbuser = "acl_user", *dbpass = "111111";
	acl::db_handle* db = new acl::db_mysql(dbaddr, dbname, dbuser, dbpass);

	if (db->open() == false)
	{
		printf("open mysql db error\r\n");
		delete db;
		return NULL;
	}
	return db;
}
/**
 * 打开 sqlite 数据库句柄
 * @return {acl::db_handle*} 返回值为 NULL 表示连接数据库失败
 */
static acl::db_handle* open_sqlite(void)
{
	const char* dbfile = "test.db";
	acl::db_handle* db = new acl::db_sqlite(dbfile);

	if (db->open() == flase)
	{
		printf("open mysql db error\r\n");
		delete db;
		return NULL;
	}
	return db;
}
////////////////////////////////////////////////////////////////////////////////
void db_demo(void)
{
	acl::db_handle* db;

	// 操作 mysql 数据库过程
	db = open_mysql();
	if (db)
	{
		tbl_select(*db);
		delete db;
	}

	// 操作 sqlite 数据库过程
	db = open_sqlite();
	if (db)
	{
		tbl_select(*db);
		delete db;
	}
}
```

从上面的例子可以看出，虽然操作的数据库不同，但数据库查询方式却是完全一样的，因为 acl 类内部屏蔽了数据库操作的差异性。下面还有几点需要注意：

- 对于数据库查询结果集有多种操作方式，开发者可以根据需要进行选择；
- 其中生成的 sql 查询语句比较简单，所以没有做特殊字符转义，真正生产环境中开发者应注意对 sql 中的一些变化查询字段进行转义（可以使用 acl::db_handle 类中的 escape_string 方法），以防止 sql 注入攻击；
- 如果查询的数据库结果集非空，则在处理结果完毕毕竟调用 acl::db_handle 类中的 free_result() 方法释放中间动态分配的内存；
- 在使用 acl 数据库类编写代码时不需要包含 mysql 和 sqlite 的头文件，但在程序连接阶段必须将 mysql/sqlite 的静态库加上。

## 二、数据库连接池
为了避免建立数据库连接开销对数据造成冲击，一般的数据库连接都建议使用连接池方式（尤其是在JAVA、PHP等应用中）；连接池在保持与数据库的长连接过程中，必须要处理连接中断的重连情况，使上层使用者忽略连接中断的情况。

下图为 acl 的数据库连接池中各类的继承关系及连接池基础类的函数接口：

从 mysql_pool.hpp/sqlite_pool.hpp 头文件中可以看出，二者的主要区别是构造函数略有不同：
db_pool 类为数据库连接池基类，其中主要有两个方法：

```c++
 	/**
	 * 从数据库中连接池获得一个数据库连接，该函数返回的数据库
	 * 连接对象用完后必须调用 db_pool->put(db_handle*) 将连接
	 * 归还至数据库连接池，由该函数获得的连接句柄不能 delete，
	 * 否则会造成连接池的内部计数器出错
	 * @return {db_handle*} 返回空，则表示出错
	 */
	db_handle* peek();

	/**
	 * 将数据库连接放回至连接池中，当从数据库连接池中获得连接
	 * 句柄用完后应该通过该函数放回，不能直接 delete，因为那样
	 * 会导致连接池的内部记数发生错误
	 * @param conn {db_handle*} 数据库连接句柄，该连接句柄可以
	 *  是由 peek 创建的，也可以单独动态创建的
	 * @param keep {bool} 归还给连接池的数据库连接句柄是否继续
	 *  保持连接，如果否，则内部会自动删除该连接句柄
	 */
	void put(db_handle* conn, bool keep = true);
```

mysql 数据库连接池的构造函数如下：

```c++
	/**
	 * 采用 mysql 数据库时的构造函数
	 * @param dbaddr {const char*} mysql 服务器地址，格式：IP:PORT，
	 *  在 UNIX 平台下可以为 UNIX 域套接口
	 * @param dbname {const char*} 数据库名
	 * @param dbuser {const char*} 数据库用户
	 * @param dbpass {const char*} 数据库用户密码
	 * @param dblimit {int} 数据库连接池的最大连接数限制
	 * @param dbflags {unsigned long} mysql 标记位
	 * @param auto_commit {bool} 是否自动提交
	 * @param conn_timeout {int} 连接数据库超时时间(秒)
	 * @param rw_timeout {int} 与数据库通信时的IO时间(秒)
	 */
	mysql_pool(const char* dbaddr, const char* dbname,
		const char* dbuser, const char* dbpass,
		int dblimit = 64, unsigned long dbflags = 0,
		bool auto_commit = true, int conn_timeout = 60,
		int rw_timeout = 60);
```

下面以 mysql 为例写一个简单的使用连接池的函数：

```c++
void dbpool_demo(void)
{
	const char* dbaddr = "127.0.0.1:3306";
	const char* dbname = "acl_test_db";
	const char* dbuser = "acl_user", *dbpass = "111111";
	acl::db_pool* dbp = new acl::mysql_pool(dbaddr, dbname, dbuser, dbpass);  // 创建 mysql 连接池
	acl::db_handle* dbh = dbp->peek(); // 从连接池中获取一个数据库连接
	if (dbh == NULL)
	{
		printf("peek db connection error\r\n");
		delete dbp;
		return;
	}
	tbl_select(*dbh);  // 从数据库中查询数据（使用上面的查询例子）
	dbh->put(dbh);	// 归还数据库连接给连接池
	delete dbh; // 删除连接池对象
}
```

由上面示例可以看出 acl 中的数据库连接池还是比较简单易用的，不过需要注意以下几点：

- 在创建数据库连接池对象时并不立刻连接后端的数据库，数据库的连接过程一般发生在 acl::db_pool::peek() 过程，但在调用 peek 时如果连接池有可用连接则直接使用之；
- 在使用数据库连接操作数据库时，如果因为网络意外导致连接断开，内部会根据数据库连接的返回错误号决定是否需要重试该数据库操作；
- 在用完数据库连接后需要调用 acl::db_pool::put() 过程归还数据库连；
- 在编译 lib_acl_cpp 库时必须需要指定 Makefile.db 为工程文件(make -f Makefile.db)，这样才能使 lib_acl_cpp.a 库内部的数据库功能生效；同时在编译自己的应用程序时必须指定 libmysqlclient_r.a 的链接位置。

好了，关于如何使用 acl 库编写数据库应用先写到此，欢迎读者批评指正。

其它有关数据库使用例子请参考：
- acl\lib_acl_cpp\samples\mysql
- acl\lib_acl_cpp\samples\sqlite