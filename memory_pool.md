---
title: 使用 acl 内存池模块管理动态对象
date: 2015-11-15 00:27
categories: 内存管理
---

C/C++ 最容易出问题的地方是内存管理，容易造成内存泄露和内存越界，这一直是 C/C++ 程序员比较头疼的事情，但 C/C++ 最大的优势也是内存管理，可以让程序员直接管理内存，从而使程序运行更为高效。acl 库中的内存池管理器 dbuf_guard 在管理内存的效率上比系统级内存管理方式（malloc/free, new/delete）更为高效，同时也使内存管理更为健壮性，这可以使 C/C++ 程序员避免出现一些常见的内存问题。本节主要介绍了 acl 库中 dbuf_guard 类的设计特点及使用方法。

dbuf_guard 类内部封装了内存池对象类 dbuf_pool，提供了与 dbuf_pool 一样的内存分配方法，dbuf_guard 类与 dbuf_pool 类的最大区别是 dbuf_guard 类对象既可以在堆上动态分配，也可以在栈上分配，而 dbuf_pool 类对象必须在堆上动态分配，另外，dbuf_guard 类还统一管理在 dbuf_pool 内存池上所创建的所有 dbuf_obj 子类对象，在 dbuf_guard 类对象析构时，所有由其管理的 dbuf_obj 子类对象被统一析构。下面先从一个简单的例子来说明 dbuf_guard 的使用方法，例子如下：

```c++
void test(void)
{
	// 定义一个内存池管理对象
	acl::dbuf_guard dbuf;

#define	STR	"hello world"

	// 在 dbuf 对象上动态创建一个字符串内存
	char* str = dbuf.dbuf_strdup(STR);
	printf("str: %s\r\n", str);

	// 在 dbuf 对象上动态创建内存

	// sizeof(STR) 取得 STR 所包含的字符串长度 + 1
	str = (char*) dbuf.dbuf_alloc(sizeof(STR));
	memcpy(str, STR, sizeof(STR));
	printf("str: %s\r\n", str);

	// 在本函数返回前，dbuf 对象自动被销毁，同时释放其所管理的
	// 内存池以及在内存池上分配的内存
}
```

上面的例子展示了使用 dbuf_guard 类对象直接分配内存块的过程，可以看出，所有在 dbuf_guard 对象的内存池上动态分配的内存都会在 dbuf_guard 对象销毁时被自动释放。这只是一个简单的使用 dbuf_guard 进行内存管理的例子，那么对于 C++ 对象的内存如何进行管理呢？请看下面的的例子：

```c++
class myobj : public acl::dbuf_obj
{
public:
	myobj()
	{
		// 调用系统 API 分配动态内存
		str_ = strdup("hello world");
	}

	void run()
	{
		printf("str: %s\r\n", str_);
	}

private:
	char* str_;

	~myobj()
	{
		// 释放构造函数中分配的内存，否则会造成内存泄露
		free(str_);
	}
};

void test(void)
{
	acl::dbuf_guard dbuf;

	// 调用 dbuf_guard::create<T>() 模板函数创建 myobj 对象
	myobj* obj = dbuf.create<myobj>();

	// 调用 myobj::run 方法
	obj->run();

	// 本函数返回前，dbuf 对象被销毁，obj 一起被销毁
}
```

该例子展示了 C++ 对象在 dbuf_guard 动态创建的过程，其中有两个要点：

- 由 dbuf_guard 对象统一管理的 C++ 对象必须是 dbuf_obj 的子类，这样在 dbuf_guard 类对象的析构时才可以通过调用 dbuf_obj 的析构函数来达到将 dbuf_obj 子类析构的目的（其中 dbuf_obj 的析构函数为虚函数）；
- 创建 dbuf_obj 的子类对象时，调用 dbuf_guard 对象的模板函数 create，同时指定子类名称 myobj 来创建 myobj 对象，create 内部会自动将该对象指针加入内部数组对象集合中，从而达到统一管理的目的。

上面例子的构造函数不带参数，在 dbuf_guard::create 模板函数中允许添加 0  至 10 个参数（其实内部有 11 个 create 模板函数），如果一个构造函数需要多于 10 个参数，则说明该构造函数设计的太复杂了。下面的例子展示了构造函数带参数的类对象创建过程：

```c++
// 继承于 acl::dbuf_obj 类
class myobj : public acl::dbuf_obj
{
public:
	myobj(int i, int j)
	{
		printf("myobj, i=%d, j=%d\r\n", i, j);
		// 在内存池上分配动态内存
		str_ = dbuf.dbuf_strdup("hello world");
	}

	void run()
	{
		printf("str: %s\r\n", str_);
	}

private:
	char* str_;

	~myobj()
	{
		// 因为 str_ 的内存也是创建在内存池上，所以此处禁止再调用 free 来释放该内存
		// free(str_);
	}
};

void test(void)
{
	acl::dbuf_guard dbuf;

	// 调用 dbuf_guard::create<T>(P1, P2) 模板函数创建构造函数带两个参数的 myobj 对象
	myobj* obj = dbuf.create<myobj>(10, 100);

	// 调用 myobj::run 方法
	obj->run();

	// 本函数返回前，dbuf 对象被销毁，obj 一起被销毁
}
```

在 dbuf_guard 类中除了上面提供的方法外，还提供了以下多个方法方便使用：

```c++
	/**
	 * 调用 dbuf_pool::dbuf_reset
	 * @param reserve {size_t}
	 * @return {bool}
	 */
	bool dbuf_reset(size_t reserve = 0)
	{
		return dbuf_->dbuf_reset(reserve);
	}

	/**
	 * 调用 dbuf_pool::dbuf_alloc
	 * @param len {size_t}
	 * @return {void*}
	 */
	void* dbuf_alloc(size_t len)
	{
		return dbuf_->dbuf_alloc(len);
	}

	/**
	 * 调用 dbuf_pool::dbuf_calloc
	 * @param len {size_t}
	 * @return {void*}
	 */
	void* dbuf_calloc(size_t len)
	{
		return dbuf_->dbuf_calloc(len);
	}

	/**
	 * 调用 dbuf_pool::dbuf_strdup
	 * @param s {const char*}
	 * @return {char*}
	 */
	char* dbuf_strdup(const char* s)
	{
		return dbuf_->dbuf_strdup(s);
	}

	/**
	 * 调用 dbuf_pool::dbuf_strndup
	 * @param s {const char*}
	 * @param len {size_t}
	 * @return {char*}
	 */
	char* dbuf_strndup(const char* s, size_t len)
	{
		return dbuf_->dbuf_strndup(s, len);
	}

	/**
	 * 调用 dbuf_pool::dbuf_memdup
	 * @param addr {const void*}
	 * @param len {size_t}
	 * @return {void*}
	 */
	void* dbuf_memdup(const void* addr, size_t len)
	{
		return dbuf_->dbuf_memdup(addr, len);
	}

	/**
	 * 调用 dbuf_pool::dbuf_free
	 * @param addr {const void*}
	 * @return {bool}
	 */
	bool dbuf_free(const void* addr)
	{
		return dbuf_->dbuf_free(addr);
	}

	/**
	 * 调用 dbuf_pool::dbuf_keep
	 * @param addr {const void*}
	 * @return {bool}
	 */
	bool dbuf_keep(const void* addr)
	{
		return dbuf_->dbuf_keep(addr);
	}

	/**
	 * 调用 dbuf_pool::dbuf_unkeep
	 * @param addr {const void*}
	 * @return {bool}
	 */
	bool dbuf_unkeep(const void* addr)
	{
		return dbuf_->dbuf_unkeep(addr);
	}

	/**
	 * 获得 dbuf_pool 对象
	 * @return {acl::dbuf_pool&}
	 */
	acl::dbuf_pool& get_dbuf() const
	{
		return *dbuf_;
	}

	/**
	 * 可以手动调用本函数，将在 dbuf_pool 上分配的 dbuf_obj 子类对象交给
	 * dbuf_guard 对象统一进行销毁管理；严禁将同一个 dbuf_obj 子类对象同
	 * 时将给多个 dbuf_guard 对象进行管理，否则将会产生对象的重复释放
	 * @param obj {dbuf_obj*}
	 * @return {int} 返回 obj 被添加后其在 dbuf_obj 对象数组中的下标位置，
	 *  dbuf_guard 内部对 dbuf_obj 对象的管理具有防重添加机制，所以当多次
	 *  将同一个 dbuf_obj 对象置入同一个 dbuf_guard 对象时，内部只会放一次
	 */
	int push_back(dbuf_obj* obj);

	/**
	 * 获得当前内存池中管理的对象数量
	 * @return {size_t}
	 */
	size_t size() const
	{
		return size_;
	}

	/**
	 * 获得当前内存池中管理的对象集合，该函数必须与 size() 函数配合使用，
	 * 以免指针地址越界
	 * @return {dbuf_obj**} 返回 dbuf_obj 对象集合，永远返回非 NULL 值，
	 *  该数组的大小由 size() 函数决定
	 */
	dbuf_obj** get_objs() const
	{
		return objs_;
	}

	/**
	 * 返回指定下标的对象
	 * @param pos {size_t} 指定对象的下标位置，不应越界
	 * @return {dbuf_obj*} 当下标位置越界时返回 NULL
	 */
	dbuf_obj* operator[](size_t pos) const;

	/**
	 * 返回指定下标的对象
	 * @param pos {size_t} 指定对象的下标位置，不应越界
	 * @return {dbuf_obj*} 当下标位置越界时返回 NULL
	 */
	dbuf_obj* get(size_t pos) const;

	/**
	 * 设置内建 objs_ 数组对象每次在扩充空间时的增量，内部缺省值为 100
	 * @param incr {size_t}
	 */
	void set_increment(size_t incr);

       同时，dbuf_obj 类中也提供了额外的操作方法：

	/**
	 * 获得该对象在 dbuf_guard 中的数组中的下标位置
	 * @return {int} 返回该对象在 dbuf_guard 中的数组中的下标位置，当该
	 *  对象没有被 dbuf_guard 保存时，则返回 -1，此时有可能会造成内存泄露
	 */
	int pos() const
	{
		return pos_;
	}

	/**
	 * 返回构造函数中 dbuf_guard 对象
	 * @return {dbuf_guard*}
	 */
	dbuf_guard* get_guard() const
	{
		return guard_;
	}
```

最后，以一个稍微复杂的例子做为结尾(该例子源码在 acl 库中的位置：lib_acl_cpp\samples\dbuf\dbuf2)，该例子展示了使用 dbuf_guard 类的几种方法：

```c++
#include "stdafx.h"
#include <sys/time.h>

/**
 * dbuf_obj 子类，在 dbuf_pool 上动态分配，由 dbuf_guard 统一进行管理
 */
class myobj : public acl::dbuf_obj
{
public:
	myobj(acl::dbuf_guard* guard = NULL) : dbuf_obj(guard)
	{
		ptr_ = strdup("hello");
	}

	void run()
	{
		printf("----> run->hello world <-----\r\n");
	}

private:
	char* ptr_;

	// 将析构声明为私人，以强制要求该对象被动态分配，该析构函数将由
	// dbuf_guard 统一调用，以释放本类对象中产生的动态内存(ptr_)
	~myobj()
	{
		free(ptr_);
	}
};

static void test_dbuf(acl::dbuf_guard& dbuf)
{
	for (int i = 0; i < 102400; i++)
	{
		// 动态分配内存
		char* ptr = (char*) dbuf.dbuf_alloc(10);
		(void) ptr;
	}

	for (int i = 0; i < 102400; i++)
	{
		// 动态分配字符串
		char* str = dbuf.dbuf_strdup("hello world");
		if (i < 5)
			printf(">>str->%s\r\n", str);
	}

	// 动态分配内存

	(void) dbuf.dbuf_alloc(1024);
	(void) dbuf.dbuf_alloc(1024);
	(void) dbuf.dbuf_alloc(2048);
	(void) dbuf.dbuf_alloc(1024);
	(void) dbuf.dbuf_alloc(1024);
	(void) dbuf.dbuf_alloc(1024);
	(void) dbuf.dbuf_alloc(1024);
	(void) dbuf.dbuf_alloc(1024);
	(void) dbuf.dbuf_alloc(1024);
	(void) dbuf.dbuf_alloc(1024);
	(void) dbuf.dbuf_alloc(1024);
	(void) dbuf.dbuf_alloc(10240);

	for (int i = 0; i < 10000; i++)
	{
		// 动态分配 dbuf_obj 子类对象，并通过将 dbuf_guard 对象传入
		// dbuf_obj 的构造函数，从而将之由 dbuf_guard 统一管理，

		myobj* obj = dbuf.create<myobj>(&dbuf);

		// 验证 dbuf_obj 对象在 dbuf_guard 中的在在一致性
		assert(obj == dbuf[obj->pos()]);

		// 调用 dbuf_obj 子类对象 myobj 的函数 run
		if (i < 10)
			obj->run();
	}

	for (int i = 0; i < 10000; i++)
	{
		myobj* obj = dbuf.create<myobj>();

		assert(dbuf[obj->pos()] == obj);

		if (i < 10)
			obj->run();
	}

	for (int i = 0; i < 10000; i++)
	{
		myobj* obj = dbuf.create<myobj>(&dbuf);

		// 虽然多次将 dbuf_obj 对象置入 dbuf_guard 中，因为 dbuf_obj
		// 内部的引用计数，所以可以防止被重复添加
		(void) dbuf.push_back(obj);
		(void) dbuf.push_back(obj);
		(void) dbuf.push_back(obj);

		assert(obj == dbuf[obj->pos()]);

		if (i < 10)
			obj->run();
	}
}

static void wait_pause()
{
	printf("Enter any key to continue ...");
	fflush(stdout);
	getchar();
}

static void test1()
{
	// dbuf_gaurd 对象创建在栈上，函数返回前该对象自动销毁
	acl::dbuf_guard dbuf;

	test_dbuf(dbuf);
}

static void test2()
{
	// 动态创建 dbuf_guard 对象，需要手动销毁该对象
	acl::dbuf_guard* dbuf = new acl::dbuf_guard;

	test_dbuf(*dbuf);

	// 手工销毁该对象
	delete dbuf;
}

static void test3()
{
	// 将内存池对象 dbuf_pool 做为 dbuf_guard 构造函数参数传入，当
	// dbuf_guard 对象销毁时，dbuf_pool 对象一同被销毁
	acl::dbuf_guard dbuf(new acl::dbuf_pool);

	test_dbuf(dbuf);
}

static void test4()
{
	// 动态创建 dbuf_guard 对象，同时指定内存池中内存块的分配倍数为 10，
	// 即指定内部每个内存块大小为 4096 * 10 = 40 KB，同时
	// 指定内部动态数组的初始容量大小
	acl::dbuf_guard dbuf(10, 100);

	test_dbuf(dbuf);
}

static void test5()
{
	acl::dbuf_pool* dp = new acl::dbuf_pool;

	// 在内存池对象上动态创建 dbuf_guard 对象，这样可以将内存分配的次数
	// 进一步减少一次
	acl::dbuf_guard* dbuf = new (dp->dbuf_alloc(sizeof(acl::dbuf_guard)))
		acl::dbuf_guard(dp);

	test_dbuf(*dbuf);

	// 因为 dbuf_gaurd 对象也是在 dbuf_pool 内存池对象上动态创建的，所以
	// 只能通过直接调用 dbuf_guard 的析构函数来释放所有的内存对象；
	// 既不能直接 dbuf_pool->desotry()，也不能直接 delete dbuf_guard 来
	// 销毁 dbuf_guard 对象
	dbuf->~dbuf_guard();
}

class myobj2 : public acl::dbuf_obj
{
public:
	myobj2() {}

	void run()
	{
		printf("hello world\r\n");
	}

private:
	~myobj2() {}
};

class myobj3 : public acl::dbuf_obj
{
public:
	myobj3(int i) : i_(i) {}

	void run()
	{
		printf("hello world: %d\r\n", i_);
	}

private:
	~myobj3() {}

private:
	int i_;
};

class myobj_dummy
{
public:
	myobj_dummy() {}

	void run()
	{
		printf("can't be compiled\r\n");
	}

private:
	~myobj_dummy() {}
};

static void test6()
{
	acl::dbuf_guard dbuf;

	myobj* o = dbuf.create<myobj>();
	o->run();

	myobj* o1 = dbuf.create<myobj>(&dbuf);
	o1->run();

	myobj2* o2 = dbuf.create<myobj2>();
	o2->run();

	myobj3* o3 = dbuf.create<myobj3>(10);
	o3->run();

	for (int i = 0; i < 10; i++)
	{
		myobj3* o4 = dbuf.create<myobj3>(i);
		o4->run();
	}

	// below codes can't be compiled, because myobj_dummy isn't
	// acl::dbuf_obj's subclass
	// myobj_dummy* dummy = dbuf.create<myobj_dummy>();
	// dummy->run();
}

int main(void)
{
	acl::log::stdout_open(true);

	test1();
	wait_pause();

	test2();
	wait_pause();

	test3();
	wait_pause();

	test4();
	wait_pause();

	test5();
	wait_pause();

	test6();
	return 0;
}
```

github：https://github.com/acl-dev/acl
gitee：https://gitee.com/acl-dev/acl