---
title: Acl 网络协程框架编程指南
date: 2019-04-07 13:08:24
tags: 协程编程
categories: 协程编程
---

# Acl 网络协程框架编程指南
<!-- vim-markdown-toc GFM -->

* [摘要](#摘要)
* [一、概述](#一概述)
* [二、简单示例](#二简单示例)
* [三、编译安装](#三编译安装)
* [四、使用多核](#四使用多核)
* [五、多核同步](#五多核同步)
* [六、消息传递](#六消息传递)
* [七、HOOK API](#七HOOKAPI)
* [八、域名解析](#八域名解析)
* [九、协程共享栈](#九协程共享栈)
* [十、使第三方网络库协程化](#十使第三方网络库协程化)
* [十一、Windows 界面编程协程化](#十一Windows界面编程协程化)
* [十二、打印协程调用栈](#十二打印协程调用栈)
* [十三、检测协程死锁问题](#十三检测协程死锁问题)

<!-- vim-markdown-toc -->

## 摘要
本文主要讲述Acl网络协程框架的使用，从协程的应用场景出发，以一个简单的协程示例开始，然后逐步深入到Acl网络协程的各个使用场景及使用细节，以及需要避免的“坑”，希望能给大家带来实践上的帮助。
## 一、概述
讲到协程，大家必然会提到 Go 语言，毕竟是 Go 语言把协程的概念及使用实践普及的；但协程并不是一个新概念，我印象中在九十年代就出现了，当时一位同事还说微软推出了纤程（基本一个意思），可以创建成午上万个纤程，不必象线程那样只能创建较少的线程数量，但当时也没明白创建这么多纤程有啥用，只不过是一个上下文的快速切换协同而已。所以自己在写网络高并发服务时，主要还是以非阻塞方式来实现。
后来，Go 语言的作者之一 Russ Cox 在 2002 年左右用 C 实现了一个简单的基于协程的网络通信模型 -- libtask，但其只是一个简单的网络协程原型，还远达不到实践的要求。自从 Go 语言兴起后，很多基于 C/C++ 开发的协程库也多了起来，其中 Acl 协程库便是其中之一。
- Acl 工程地址：https://github.com/acl-dev/acl
- Acl 协程地址：https://github.com/acl-dev/acl/tree/master/lib_fiber

## 二、简单示例
下面为一个使用 Acl 库编写的简单线程的例子：
```c++
#include <acl-lib/acl_cpp/lib_acl.hpp>

class mythread : public acl::thread {
public:
	mythread(void) {}
	~mythread(void) {}
private:
	// 实现基类中纯虚方法，当线程启动时该方法将被回调
	// @override
	void* run(void) {
		for (int i = 0; i < 10; i++) {
			printf("thread-%lu: running ...\r\n", acl::thread::self());
		}
		return NULL;
	}
};

int main(void) {
	std::vector<acl::thread*> threads;
	for (int i = 0; i < 10; i++) {
		acl::thread* thr = new mythread;
		threads.push_back(thr);
		thr->start();  // 启动线程
	}
	
	for (std::vector<acl::thread*>::iterator it = threads.begin();
		it != threads.end(); ++it) {
		(*it)->wait();  // 等待线程退出
		delete *it;
	}
	return 0;	
}
```
上面线程例子非常简单，接着再给一个简单的协程的例子：
```c++
#include <acl-lib/fiber/libfiber.hpp>

class myfiber : public acl::fiber {
public:
	myfiber(void) {}
	~myfiber(void) {}
private:
	// 实现基类纯虚方法，当调用 fiber::start() 时，该方法将被调用
	// @override
	void run(void) {
		for (int i = 0; i < 10; i++) {
			printf("hello world! the fiber is %d\r\n", acl::fiber::self());
			acl::fiber::yield();  // 让出CPU运行权给其它协程
		}
	}
};

int main(void) {
	std::vector<acl::fiber*> fibers;
	for (int i = 0; i < 10; i++) {
		acl::fiber* fb = new myfiber;
		fibers.push_back(fb);
		fb->start();  // 启动一个协程
	}
	
	acl::fiber::schedule();  // 启用协程调度器
	
	for (std::vector<acl::fiber*>::iterator it = fibers.begin();
		it != fibers.end(); ++it) {
		delete *it;
	}
}
```
上面示例演示了协程的创建、启动及运行的过程，与前一个线程的例子非常相似，也很简单（**简单实用**是 Acl 库的设计目标之一）。

**协程调度其实是应用层面多个协程之间通过上下文切换形成的协作过程，如果一个协程库仅是实现了上下文切换，其实并不具备太多实用价值，当与网络事件绑定后，其价值才会显现出来**。下面一个简单的使用协程的网络服务程序：
```c++
#include <acl-lib/acl_cpp/lib_acl.hpp>
#include <acl-lib/fiber/libfiber.hpp>

// 客户端协程处理类，用来回显客户发送的内容，每一个客户端连接绑定一个独立的协程
class fiber_echo : public acl::fiber {
public:
	fiber_echo(acl::socket_stream* conn) : conn_(conn) {}

private:
	acl::socket_stream* conn_;
	~fiber_echo(void) { delete conn_; }

private:
	// @override
	void run(void) {
		char buf[8192];
		while (true) {
			// 从客户端读取数据（第三个参数为false表示不必填满整个缓冲区才返回）
			int ret = conn_->read(buf, sizeof(buf), false);
			if (ret == -1) {
				break;
			}
			// 向客户端写入读到的数据
			if (conn_->write(buf, ret) != ret) {
				break;
			}
		}
		delete this; // 自销毁动态创建的协程对象
	}
};

// 独立的协程过程，接收客户端连接，并将接收的连接与新创建的协程进行绑定
class fiber_listen : public acl::fiber {
public:
	fiber_listen(acl::server_socket& listener) : listener_(listener) {}

private:
	acl::server_socket& listener_;
	~fiber_listen(void) {}

private:
	// @override
	void run(void) {
		while (true) {
			acl::socket_stream* conn = listener_.accept();  // 等待客户端连接
			if (conn == NULL) {
				printf("accept failed: %s\r\n", acl::last_serror());
				break;
			}
			// 创建并启动单独的协程处理客户端连接
			acl::fiber* fb = new fiber_echo(conn);
			// 启动独立的客户端处理协程
			fb->start();
		}
		delete this;
	}
};

int main(void) {
	const char* addr = "127.0.0.1:8800";
	acl::server_socket listener;
	// 监听本地地址
	if (listener.open(addr) == false) {
		printf("listen %s error %s\r\n", addr, acl::last_serror());
		return 1;
	}

	// 创建并启动独立的监听协程，接受客户端连接
	acl::fiber* fb = new fiber_listen(listener);
	fb->start();

	// 启动协程调度器
	acl::fiber::schedule();
	return 0;
}
```
这是一个支持回显功能的网络协程服务器（也可以修改成线程模式）。使用协程或线程处理网络通信都可以采用**顺序思维**模式，不必象非阻塞网络编程那样复杂，同时使用协程的最大好处是可以创建大量的协程来处理网络高并发连接，而要创建大量的线程是不现实的（线程数非常多时，会导致操作系统的调度能力下降）。

在上面的示例中，启动协程调度（即调用 `acl::fiber::schedule()`）时，内部事件引擎为默认的内核级事件引擎，该方法有一个缺省的事件类型参数可以由用户修改，可选的事件引擎如下：
- **acl::FIBER_EVENT_T_KERNEL：** 内核级高效事件引擎，在 Linux 上使用 epoll，FreeBSD/MacOS 上使用 kqueue，Windows 平台上则会使用 IOCP；
- **acl::FIBER_EVENT_T_SELECT, acl::FIBER_EVENT_T_POLL：** 这两个引擎是跨平台的，基本上所有平台都支持；
- **acl::FIBER_EVENT_T_WMSG：** 当在 Windows 平台开发界面程序时可以使用该引擎，以使网络过程协程化且与界面操作过程同属一个线程空间中，从而可以轻松在网络 IO 过程中操作界面元素；
- **acl::FIBER_EVENT_T_IO_URING：** 这是在 Linux 5.1x 内核以上新增加的高效事件引擎 io_uring，与 epoll 仅能支持网络（及管道）IO相比，io_uring 引擎同时支持网络 IO 及文件 IO，实现了真正意义上的 IO 大统一，而且 io_uring 引擎为 IO 完成模型，非常容易与协程模式相结合。

## 三、编译安装
在编译前，需要先从 **github**  https://github.com/acl-dev/acl  下载源码，国内用户可以选择从 **gitee**  https://gitee.com/acl-dev/acl  下载源码。

### 3.1、Linux/Unix 平台上编译安装
在 Linux/Unix 平台上的编译非常简单，可以选择使用 make 方式或 cmake 方式进行编译。  
- **make 方式编译：**
在 acl 项目根目录下运行：**make && make packinstall**，则会自动执行编译与安装过程，安装目录默认为系统目录：libacl_all.a, libfiber_cpp.a, libfiber.a 将被拷贝至 /usr/lib/ 目录，头文件将被拷贝至 /usr/include/acl-lib/ 目录。
- **cmake 方式编译：**
在 acl 项目根目录下创建 build 目录，然后：**cd build && cmake .. && make**
- **将 acl 库加入至你的工程**（以 make 方式为例）
先在代码中加入头文件包含项：
```c++
#include <acl-lib/acl_cpp/lib_acl.hpp>
#include <acl-lib/fiber/libfiber.hpp>
```
然后修改你的 Makefile 文件，示例如下：
```Makefile
mytest: mytest.cpp
	g++ -o mytest mytest.cpp -lfiber_cpp -lacl_all -lfiber -ldl -lpthread -lz
```
**注意在 Makefile 中各个库的依赖顺序：** libfiber_cpp.a 依赖于 libacl_all.a 和 libfiber.a，其中 libacl_all.a 为 acl 的基础库，libfiber.a 为独立的 C 语言协程库（其不依赖于 libacl_all.a），libfiber_cpp.a 用 C++ 语言封装了 libfiber.a，且使用了 libacl_all.a 中的一些功能；同时，因为 libfiber.a 协程库 Hook 了系统的网络及IO API，所以 libfiber.a 需放在 libacl_all.a 后面，以便 libacl_all.a 中的 IO 调用过程被 libfiber.a 中的 API Hook 住；因此，在执行程序连接时需要明确告诉 gcc/g++ 连接顺序为：
libfiber_cpp.a, libacl_all.a, libfiber.a。

### 3.2、Windows 平台上编译
在 Windows 平台的编译也非常简单，可以用 vc2012/2013/2019 打开相应的工程文件进行编译，如：可以用 vc2019 打开 acl_cpp_vc2019.sln 工程进行编译。

**注意：** 在 Windows 平台上建议使用 vc2019 及以上版本打开 Acl 库，因为 Acl fiber 在 Windows 平台用来 Hook 系统 API 的第三方库（detours）目前不支持 vc2012/2013。

### 3.3 Mac 平台上编译
除可以使用 Unix 统一方式（命令行方式）编译外，还可以用 Xcode 打开工程文件进行编译。

### 3.4 Android 平台上编译
目前可以使用 Android Studio3.x 打开 acl\android\acl 目录下的工程文件进行编译。

### 3.5 使用 MinGW 编译
如果想要在 Windows 平台上编译 Unix 平台上的软件，可以借用 MinGW 套件进行编译，为此 Acl 库还提供了此种编译方式，但一般不建议用户使用这种编译方式，一方面是执行效率低，另一方面可能会存在不兼容性问题。

### 3.6 小结
为了保证 Acl 工程无障碍使用，本人在编译 Acl 库方面下了很大功夫，支持几乎在所有平台上使用原生编译环境进行编译使用，真正达到了一键编译。甚至为了避免因依赖第三方库而导致的编译问题（如：有的模块需要 zlib 库，有的需要 polassl 库，有的需要 mysql/postgresql/sqlite 库），将这些依赖第三方库的模块都写成动态加载第三方库的方式，毕竟不是所有人都需要这些第三方库所提供的功能。

## 四、使用多核
Acl 协程的调度过程是基于单CPU的（虽然也可以修改成多核调度，但考虑到很多原因，最终还是采用了单核调度模式），即创建一个线程，所创建的所有协程都在这个线程空间中运行。为了使用多核，充分使用CPU资源，可以创建多个线程（也可以创建多个进程），每个线程为一个独立的协程运行容器，各个线程之间的协程相互隔离，互不影响。
下面先修改一下上面的例子，改成多线程的协程方式：
```c++
#include <acl-lib/acl_cpp/lib_acl.hpp>
#include <acl-lib/fiber/libfiber.hpp>

// 客户端协程处理类，用来回显客户发送的内容，每一个客户端连接绑定一个独立的协程
class fiber_echo : public acl::fiber {
public:
	fiber_echo(acl::socket_stream* conn) : conn_(conn) {}

private:
	acl::socket_stream* conn_;
	~fiber_echo(void) { delete conn_; }

private:
	// @override
	void run(void) {
		char buf[8192];
		while (true) {
			int ret = conn_->read(buf, sizeof(buf), false);
			if (ret == -1) {
				break;
			}
			if (conn_->write(buf, ret) != ret) {
				break;
			}
		}
		delete this; // 自销毁动态创建的协程对象
	}
};

// 独立的协程过程，接收客户端连接，并将接收的连接与新创建的协程进行绑定
class fiber_listen : public acl::fiber {
public:
	fiber_listen(acl::server_socket& listener) : listener_(listener) {}

private:
	acl::server_socket& listener_;
	~fiber_listen(void) {}

private:
	// @override
	void run(void) {
		while (true) {
			acl::socket_stream* conn = listener_.accept();  // 等待客户端连接
			if (conn == NULL) {
				printf("accept failed: %s\r\n", acl::last_serror());
				break;
			}
			// 创建并启动单独的协程处理客户端连接
			acl::fiber* fb = new fiber_echo(conn);
			fb->start();
		}
		delete this;
	}
};

// 独立的线程调度类
class thread_server : public acl::thread {
public:
	thread_server(acl::server_socket& listener) : listener_(listener) {}
	~thread_server(void) {}

private:
	acl::server_socket& listener_;
	// @override
	void* run(void) {
		// 创建并启动独立的监听协程，接受客户端连接
		acl::fiber* fb = new fiber_listen(listener);
		fb->start();
		// 启动协程调度器
		acl::fiber::schedule(); // 内部处于死循环过程
		return NULL;
	}
};

int main(void) {
	const char* addr = "127.0.0.1:8800";
	acl::server_socket listener;
	// 监听本地地址
	if (listener.open(addr) == false) {
		printf("listen %s error %s\r\n", addr, acl::last_serror());
		return 1;
	}

	std::vector<acl::thread*> threads;
	// 创建多个独立的线程对象，每个线程启用独立的协程调度过程
	for (int i = 0; i < 4; i++) {
		acl::thread* thr = thread_server(listener);
		threads.push_back(thr);
		thr->start();
	}
	for (std::vector<acl::thread*>::iterator it = threads.begin();
		it != threads.end(); ++it) {
		(*it)->wait();
		delete *it;
	}
	return 0;
}
```
经过修改，上面的例子不仅可以支持大并发，而且还可以使用多核。

## 五、多核同步
上面的例子中涉及到了通过创建多线程使用多核的过程，但肯定会有人问，在多个线程中的协程之间如果想要共享某个资源怎么办？Acl 协程库提供了可以跨线程使用同步原语：线程协程互斥锁及条件变量。
首先介绍一下协程互斥锁类：acl::fiber_mutex，该类提供了三个方法：
```c++
	/**
	 * 等待互斥锁
	 * @return {bool} 返回 true 表示加锁成功，否则表示内部出错
	 */
	bool lock(void);

	/**
	 * 尝试等待互斥锁
	 * @return {bool} 返回 true 表示加锁成功，否则表示锁正在被占用
	 */
	bool trylock(void);

	/**
	 * 互斥锁拥有者释放锁并通知等待者
	 * @return {bool} 返回 true 表示通知成功，否则表示内部出错
	 */
	bool unlock(void);
```

下面给出一个例子，看看在多个线程中的协程之间如何进行互斥的：
```c++
#include <acl-lib/acl_cpp/lib_acl.hpp>
#include <acl-lib/fiber/libfiber.hpp>

class myfiber : public acl::fiber {
public:
	myfiber(acl::fiber_mutex& lock, int& count)
	: lock_(lock)
	, count_(count)
	{}

private:
	~myfiber(void) {}

private:
	// @override
	void run(void) {
		for (int i = 0; i < 100; i++) {
			lock_.lock();
			count_++;
			acl::fiber::delay(1);  // 本协程休息1毫秒
			lock_.unlock();
		}
		delete this;
	}
private:
	acl::fiber_mutex& lock_;
	int& count_;
};

class mythread : public acl::thread {
public:
	mythread(acl::fiber_mutex& lock, int& count)
	: lock_(lock)
	, count_(count)
	{}

	~mythread(void) {}
private:
	// @override
	void* run(void) {
		for (int i = 0; i < 100; i++) {
			acl::fiber* fb = new myfiber(lock_, count_);
			fb->start();
		}
		acl::fiber::schedule();
		return NULL;
	}
private:
	acl::fiber_mutex& lock_;
	int& count_;
};

int main(void) {
	acl::fiber_mutex lock;  // 可以用在多个线程之间、各个线程中的协程之间的同步过程
	int count = 0;

	std::vector<acl::thread*> threads;
	for (int i = 0; i < 4; i++) {
		acl::thread* thr = new mythread(lock, count);
		threads.push_back(thr);
		thr->start();
	}

	for (std::vector<acl::thread*>::iterator it = threads.begin();
		it != threads.end(); ++it) {
		(*it)->wait();
		delete *it;
	}

	printf("all over, count=%d\r\n", count);
	return 0;
}
```
**acl::fiber_mutex** 常被用在不同线程中的协程之间的同步，也可以用在多个线程之间甚至协程与独立线程之间的同步，这在很大程度弥补了 Acl 协程框架在使用多核上的不足。

## 六、消息传递
通过组合 **acl::fiber_mutex**（协程互斥锁）和 **acl::fiber_cond**（协程条件变量），实现了协程间进行消息传递的功能模块：**acl::fiber_tbox**，**fiber_tbox** 不仅可以用在同一线程内的协程之间传递消息，还可以用在不同线程中的协程之间，不同线程之间，线程与协程之间传递消息。**fiber_tbox** 为模板类，因而可以传递各种类型对象。以下给出一个示例：
```c++
#include <acl-lib/acl_cpp/lib_acl.hpp>
#include <acl-lib/fiber/libfiber.hpp>

class myobj {
public:
	myobj(void) : i_(0) {}
	~myobj(void) {}

	void set(int i) {
		i_ = i;
	}
	void test(void) {
		printf("hello world, i=%d\r\n", i_);
	}

private:
	int i_;
};

// 消费者协程，从消息管道中读取消息
class fiber_consumer : public acl::fiber {
public:
	fiber_consumer(acl::fiber_tbox<myobj>& box) : box_(box) {}

private:
	~fiber_consumer(void) {}

private:
	acl::fiber_tbox<myobj>& box_;

	// @override
	void run(void) {
		while (true) {
			myobj* o = box_.pop();
			// 如果读到空消息，则结束
			if (o == NULL) {
				break;
			}
			o->test();
			delete o;
		}
		delete this;
	}
};

// 生产者协程，向消息管道中放置消息
class fiber_producer : public acl::fiber {
public:
	fiber_producer(acl::fiber_tbox<myobj>& box) : box_(box) {}

private:
	~fiber_producer(void) {}

private:
	acl::fiber_tbox<myobj>& box_;

	// @override
	void run(void) {
		for (int i = 0; i < 10; i++) {
			myobj* o = new myobj;
			o->set(i);
			// 向消息管道中放置消息
			box_.push(o);
		}
		// 放置空消息至消息管道中，从而通知消费者协程结束
		box_.push(NULL);
		delete this;
	}
};

int main(void) {
	acl::fiber_tbox<myobj> box;
	// 创建并启动消费者协程
	acl::fiber* consumer = new fiber_consumer(box);
	consumer->start();
	// 创建并启动生产者协程
	acl::fiber* producer = new fiber_producer(box);
	producer->start();
	// 启动协程调度器
	acl::fiber::schedule();
	return 0; 
}
```
上面例子展示了同一线程中的两个协程之间的消息传递过程，因为 acl::fiber_tbox 是可以跨线程的，所以它的更大价值是用在多个线程中的不同协程之间进行消息传递。

```c++
#include <acl-lib/acl_cpp/lib_acl.hpp>
#include <acl-lib/fiber/libfiber.hpp>

class myobj {
public:
	myobj(void) : i_(0) {}
	~myobj(void) {}
	void set(int i) {
		i_ = i;
	}
	void test(void) {
		printf("hello world, i=%d\r\n", i_);
	}

private:
	int i_;
};

// 消费者协程，从消息管道中读取消息
class fiber_consumer : public acl::fiber {
public:
	fiber_consumer(acl::fiber_tbox<myobj>& box) : box_(box) {}

private:
	~fiber_consumer(void) {}

private:
	acl::fiber_tbox<myobj>& box_;

	// @override
	void run(void) {
		while (true) {
			myobj* o = box_.pop();
			// 如果读到空消息，则结束
			if (o == NULL) {
				break;
			}
			o->test();
			delete o;
		}
		delete this;
	}
};

// 生产者线程，向消息管道中放置消息
class thread_producer : public acl::thread {
public:
	thread_producer(acl::fiber_tbox<myobj>& box) : box_(box) {}
	~thread_producer(void) {}

private:
	acl::fiber_tbox<myobj>& box_;
	void* run(void) {
		for (int i = 0; i < 10; i++) {
			myobj* o = new myobj;
			o->set(i);
			box_.push(o);
		}
		box_.push(NULL);
		return NULL;
	}
};

int main(void) {
	acl::fiber_tbox<myobj> box;
	// 创建并启动消费者协程
	acl::fiber* consumer = new fiber_consumer(box);
	consumer->start();
	// 创建并启动生产者线程
	acl::thread* producer = new thread_producer(box);
	producer->start();
	// 启动协程调度器
	acl::fiber::schedule();
	// schedule() 过程返回后，表示该协程调度器结束。
	// 等待生产者线程退出
	producer->wait();
	delete producer;
	return 0; 
}
```
在该示例中，生产者为一个独立的线程，消费者为另一个线程中的协程，二者通过 acl::fiber_tbox 进行消息通信。

下面再给一个应用场景的例子，也是我们平时经常会遇到的。
```c++
#include <unistd.h>
#include <acl-lib/acl_cpp/lib_acl.hpp>
#include <acl-lib/fiber/libfiber.hpp>

class mythread : public acl::thread {
public:
	mythread(acl::fiber_tbox<int>& box) :box_(box) {}
	~mythread(void) {}

private:
	acl::fiber_tbox<int>& box_;

	// @override
	void* run(void) {
		int i;
		for (i = 0; i < 5; i++) {
			/* 假设这是一个比较耗时的操作*/
			printf("sleep one second\r\n");
			sleep(1);
		}
		int* n = new int(i);
		// 将计算结果通过消息管道传递给等待者协程
		box_.push(n);
		return NULL; 
	}
};

class myfiber : public acl::fiber {
public:
	myfiber(void) {}
	~myfiber(void) {}

private:
	// @override
	void run(void) {
		acl::fiber_tbox<int> box;
		mythread thread(box);
		thread.set_detachable(true);
		thread.start();  // 启动独立的线程计算耗时运算
		int* n = box.pop();  // 等待计算线程返回运算结果，仅会阻塞当前协程
		printf("n is %d\r\n", *n);
		delete n;
	}
};

int main(void) {
	myfiber fb;
	fb.start();
	acl::fiber::schedule();
	return 0;
}
```

协程一般用在网络高并发环境中，但协程并不是万金油，协程并不适合计算密集型应用，因为线程才是操作系统的最小调度单元，而协程不是，所以当遇到一些比较耗时的运算时，为了不阻塞当前协程所在的协程调度器，应将该耗时运算过程中抛给独立的线程去处理，然后通过 **acl::fiber_tbox** 等待线程的运算结果。

## 七、HOOK API
为了使现有的很多网络应用和网络库在尽量不修改的情况下协程化，Acl 协程库 Hook 了很多与 IO 和网络通信相关的系统 API，目前已经 Hook 的系统 API 有：
| 内容项 |API|
|--|--|
|网络相关|socket/listen/accept/connect  |
|IO相关|read/readv/recv/recvfrom/recvmsg/write/writev/send/sendto/sendmsg/sendfile64|
|域名相关|gethostbyname(_r)/getaddrinfo/freeaddrinfo|
|事件相关|select/poll/epoll_create/ epoll_ctl/epoll_wait|
|其它|close/sleep|

## 八、域名解析
使用协程方式编写网络通信程序，域名解析是不能绕过的，记得有一个协程库说为了支持域名解析，甚至看了相关实现代码，然后说通过 Hook _poll API 就可以了，实际上这并不是通用的做法，至少在我的环境里通过 Hook _poll API 是没用的，所以最稳妥的做法还是应该将 DNS 查询协议实现了，在 acl 的协程库中，域名解析模块在初期集成了第三方 DNS 库，参见：https://github.com/wahern/dns  ，但因为该第 DNS 库存在很多问题（如：不能跨平台，代码混乱，没处理IO异常等问题），所以最终 acl 协程丢弃该库，自己实现了一套更加方便灵活跨平台的 DNS 协议库。 

下面给出协程下进行域名解析的例子：
```c++
#include <netdb.h>
#include <arpa/inet.h>
#include <stdio.h>
#include <stdlib.h>
#include <acl-lib/fiber/libfiber.hpp>

class ns_lookup : public acl::fiber {
public:
    ns_lookup(const char* name) : name_(name) {}

private:
    ~ns_lookup(void) {}

private:
    std::string name_;

    // @override
    void run(void) {
        struct hostent *ent = gethostbyname(name_.c_str());
        if (ent == NULL) {
            printf("gethostbyname error=%s, name=%s\r\n",
                hstrerror(h_errno), name_.c_str());
            delete this;
            return;
        }

        printf("h_name=%s, h_length=%d, h_addrtype=%d\r\n",
            ent->h_name, ent->h_length, ent->h_addrtype);
        for (int i = 0; ent->h_addr_list[i]; i++) {
            char *addr = ent->h_addr_list[i];
            char  ip[64];
            const char *ptr;

            ptr = inet_ntop(ent->h_addrtype, addr, ip, sizeof(ip));
            printf(">>>addr: %s\r\n", ptr);
        }

        delete this;
    }
};

int main(void) {
    const char *name1 = "www.google.com", *name2 = "www.baidu.com",
               *name3 = "zsx.xsz.zzz";

    acl::fiber* fb = new ns_lookup(name1);
    fb->start();

    fb = new ns_lookup(name2);
    fb->start();

    fb = new ns_lookup(name3);
    fb->start();

    acl::fiber::schedule();
    return 0;
}
```
上面例子中，在协程 `ns_lookup` 中调用 `gethostbyname` API 进行域名查询与我们平时进行域名解析没有什么不同，在 Acl 协程中 Hook 了 `gethostbyname(_r)` 等域名解析相关 API 并进行了协程化处理，对于用户而言是无感知的。编译运行上面例子，便可得到如下结果：
```text
gethostbyname error=No address associated with name, name=zsx.xsz.zzz
h_name=www.a.shifen.com, h_length=4, h_addrtype=2
>>>addr: 110.242.68.4
>>>addr: 110.242.68.3
h_name=www.google.com, h_length=4, h_addrtype=2
>>>addr: 199.59.149.210
```

## 九、协程共享栈
Acl 协程是有协程栈的，在网络高并发时具有很高的网络处理能力，但有栈协程在非常大量的并发时势必占用较多的内存，为此在 Acl 协程中增加了对于**共享运行栈**的支持（据说该方式在微信后端服务使用后可以大幅节省内存占用）。

以下给出了使用协程共享栈的使用方法：
```c++
class myfiber : public acl::fiber {
public:
	myfiber(void) {}

private:
	~myfiber(void) {}

	// @override
	void run(void) {
		// ...
		delete this;
	}
}

void start(void) {
	for (int i = 0; i < 100000; i++) {
		acl::fiber* fb = new myfiber;
		fb->start(8000, true);  // 第二个参数为 true 时表示该协程使用共享栈模式
	}

	acl::fiber::schedule();
}
```

从上面示例可以看出，我们只需在调用协程启动方法 `start()` 时将第二个参数设为 `true` 便可以使该协程以 `协程共享运行栈` 模式运行；在 Acl 协程调度器中`协程独立运行栈`模式和`共享运行栈`模式是可以共存的（即：可以同时存在）。

在实践中这的确可以大幅减少高并发时的内存使用，虽然进行栈拷贝时会耗费一些时间，但整体影响并不太大。

相对于栈拷贝时的时间损耗，在使用共享栈方式编程时有一点需要**特别注意：** 创建在栈上的变量不能在协程之间或协程与线程之间共享，即是说，一个协程 F1 中的变量 A 传递给另一个协程 F2，并等待 F2 处理后返回，此时的 A 变量不能被创建在 F1 的栈上，因为运行栈在由 F1 切换到 F2 时，变量 A 的地址空间“暂时消失了”，此时变成了 F2 的栈空间，如果该变量在 F2 中继续被使用的话，就会存在地址非法使用的问题；解决变量在协程间共享的方法是将变量创建在堆上（即用 malloc 或 new 创建）。

**注意：** 当前 Acl 协程`共享运行栈`模式仅支持 Unix 平台，即还不支持 Windows 平台。

下面给出一个例子，如果在**非共享运行栈**模式运行是没有问题的，但如果以**共享栈**方式运行则就会出现问题：
```c++
#include <acl-lib/acl_cpp/lib_acl.hpp>
#include <acl-lib/fiber/libfiber.hpp>

class fiber_runner : public acl::fiber {
public:
	fiber_runner(acl::fiber_tbox<int>* box) : box_(box) {}

private:
	~fiber_runner(void) {}

private:
	acl::fiber_tbox<int>* box_(box);

	// @override
	void run(void) {
		int* n = new int;
		*n = 1000;
		box_.push(n);

		delete this;
	}
};

class fiber_waiter : public acl::fiber {
public:
	fiber_waiter(bool share_stack) : share_stack_(share_stack) {}

private:
	~fiber_waiter(void) {}

private:
	bool share_stack_;

	// @override
	void run(void) {
		acl::fiber_tbox<int> box;  // box 创建在栈上，且在两个协程之间共享
		acl::fiber* fb = new fiber_runner(&box);
		fb->start(64000, share_stack_);
		int * n = box.pop();
		delete n;
		delete this;
	}
};

int main(void) {
	bool share_stack = false;

	acl::fiber* fb = new fiber_waiter(share_stack);
	fb->start(64000, share_stack);

	acl::fiber::schedule();
	return 0;
}
```
在上面的例子中，`box` 对象在两个协程之间共享使用，因为两个协程以`协程独立栈`方式运行（上面的 share_stack = false），所以不会有问题，但如果将控制参数 `share_stack = true` 则两个协程都运行在 `共享栈` 上，此时就会产生问题：在协程 `fiber_waiter` 中创建协程 `fiber_runner` 并将创建在运行栈上的变量 `box` 传递给 `fiber_runner` 对象后，调用 `box.pop()`，此时协程 `fiber_waiter` 就会被挂起（其运行栈从共享栈中拷贝并保存起来）并让出 `协程共享运行栈` 给协程 `fiber_runner`，在协程 `fiber_runner` 内部操作由协程 `fiber_waiter` 传递来的变量 `box` 时，创建在`共享运行栈`上的对象 `box` 的地址空间可能已经被协程 `fiber_runner` 的栈数据覆盖了，所以传递到 `fiber_runner` 中的对象 box 的地址上的数据发生了改变（不再是 box 对象了），这时如果再操作 box 对象便是非法了。

要想解决在 `协程共享运行栈` 模式下地址空间覆盖时带来的问题，就需要在协程之间共享的对象创建在堆上，针对上面的例子，可修改成如下方式：
```c++
class fiber_waiter : public acl::fiber {
	//...
private:
	// @override
	void run(void) {
		// 在两个协程之间共享的对象需要创建在堆上
		acl::fiber_tbox<int>* box = new acl::fiber_tbox<int>;
		acl::fiber* fb = new fiber_runner(box);
		fb->start(64000, share_stack_);
		int *n = box->pop();
		delete n;
		delete box;
		delete this;
	}
};
```

针对上面的例子，如果协程`fiber_waiter`以独立栈方式运行，而协程`fiber_runner`以共享栈方式运行时，box 对象能否创建栈上呢？答案是肯定的，因为当协程`fiber_waiter`被挂起时，其栈空间依然保留不会被其它协程占用，所以在其栈创建的对象的地址上的数据不会被覆盖。

## 十、使第三方网络库协程化
通常网络通信库都是阻塞式的，因为非阻塞式的通信库的通用性不高（使用各自封装的事件引擎，很难达到应用层的使用一致性），如果把这些第三方通信库（如：mysql 客户端库，Acl 中的 Redis 库等）使用协程所提供的 IO 及网络  API 重写一遍则工作量太大，不太现实，所以在 Acl 协程库中通过 Hook 系统 API，使阻塞式网络通信库协程化变得简单。所谓网络库协程化就是使这些网络库可以应用在协程环境中，从而可以很容易编写出支持高并发的网络程序。

先写一个将 Acl Redis 客户端库协程化的例子：

```c++
#include <acl-lib/acl_cpp/lib_acl.hpp>
#include <acl-lib/fiber/libfiber.hpp>

class fiber_redis : public acl::fiber {
public:
	fiber_redis(acl::redis_client_cluster& cluster) : cluster_(cluster) {}

private:
	~fiber_redis(void) {}

private:
	acl::redis_client_cluster& cluster_;
	// @override
	void run(void) {
		const char* key = "hash-key";
		for (int i = 0; i < 100; i++) {
			acl::redis cmd(&cluster_);
			acl::string name, val;
			name.format("hash-name-%d", i);
			val.format("hash-val-%d", i);
			if (cmd.hset(key, name, val) == -1) {
				printf("hset error: %s, key=%s, name=%s\r\n",
					cmd.result_error(), key, name.c_str());
				break;
			}
		}
		delete this;
	}
};

int main(void) {
	const char* redis_addr = "127.0.0.1:6379";
	acl::redis_client_cluster cluster;
	cluster.set(redis_addr, 0);
	for (int i = 0; i < 100; i++) {
		acl::fiber* fb = new fiber_redis(cluster);
		fb->start();
	}
	acl::fiber::schedule();
	return 0;
}
```
读者可以尝试将上面的代码拷贝到自己机器上，编译后运行一下。另外，这个例子是只有一个线程，所以会发现 acl::redis_client_cluster 的使用方式和在线程下是一样的。如果将 acl::redis_client_cluster 在多个线程调度器上共享会怎样？还是有一点区别，如下：

```c++
#include <acl-lib/acl_cpp/lib_acl.hpp>
#include <acl-lib/fiber/libfiber.hpp>

// 每个协程共享相同的 cluster 对象，向 redis-server 中添加数据
class fiber_redis : public acl::fiber {
public:
	fiber_redis(acl::redis_client_cluster& cluster) : cluster_(cluster) {}
private:
	~fiber_redis(void) {}
private:
	acl::redis_client_cluster& cluster_;
	// @override
	void run(void) {
		const char* key = "hash-key";
		for (int i = 0; i < 100; i++) {
			acl::redis cmd(&cluster_);
			acl::string name, val;
			name.format("hash-name-%d", i);
			val.format("hash-val-%d", i);
			if (cmd.hset(key, name, val) == -1) {
				printf("hset error: %s, key=%s, name=%s\r\n",
					cmd.result_error(), key, name.c_str());
				break;
			}
		}
		delete this;
	}
};

// 每个线程运行一个独立的协程调度器
class mythread : public acl::thread {
public:
	mythread(acl::redis_client_cluster& cluster) : cluster_(cluster) {}
	~mythread(void) {}
private:
	acl::redis_client_cluster& cluster_;
	// @override
	void* run(void) {
		for (int i = 0; i < 100; i++) {
			acl::fiber* fb = new fiber_redis(cluster_	);
			fb->start();
		}
		acl::fiber::schedule();
		return NULL;
	}
};

int main(void) {
	const char* redis_addr = "127.0.0.1:6379";
	acl::redis_client_cluster cluster;
	cluster.set(redis_addr, 0);
	cluster.bind_thread(true);

	// 创建多个线程，共享 redis 集群连接池管理对象：cluster，即所有线程中的
	// 所有协程共享同一个 cluster 集群管理对象
	std::vector<acl::thread*> threads;
	for (int i = 0; i < 4; i++) {
		acl::thread* thr = new mythread(cluster);
		threads.push_back(thr);
		thr->start();
	}
	for (std::vector<acl::thread*>::iterator it = threads.begin();
		it != threads.end(); ++it) {
		(*it)->wait();
		delete *it;
	}
	return 0;
}
```
在这个多线程多协程环境里使用 acl::redis_client_cluster 对象时与前面的一个例子有所不同，在这里调用了：**cluster.bind_thread(true);** 

为何要这样做？原因是 Acl Redis 的协程调度器是单线程工作模式，网络套接字句柄在协程环境里不能跨线程使用，当调用 bind_thread(true) 后，Acl 连接池管理对象会自动给每个线程分配一个连接池对象，每个线程内的所有协程共享这个绑定于本线程的连接池对象。

## 十一、Windows 界面编程协程化
在Windows下写过界面程序的程序员都经历过使通信模块与界面结合的痛苦过程，因为 Windows 界面过程是基于 win32 消息引擎驱动的，所以在编写通信模块时一般有两个选择：要么使用 Windows 提供的异步非阻塞 API，要么把通信模块放在独立于界面的单独线程中然后通过窗口消息将结果通知窗口界面过程。
Acl 协程库的事件引擎支持 win32 消息引擎，所以很容易将界面过程的通信过程协程化，采用这种方式，一方面程序员依然可以采用顺序编程方式，另一方面通信协程与界面过程运行于相同的线程空间，则二者在相互访问对方的成员对象时不必加锁，从而使编写通信过程变得更加简单。
下面以一个简单的对话框为例说明界面网络通信协程化过程：    
1. 首先使用向导程序生成一个对话框界面程序，需要指定支持 socket 通信；
2. 然后在 OnInitDialog() 方法尾部添加如下代码：
```c++
	// 设置协程调度的事件引擎，同时将协程调度设为自动启动模式
	acl::fiber::init(acl::FIBER_EVENT_T_WMSG, true);
	// HOOK ACL 库中的网络 IO 过程
	acl::fiber::acl_io_hook();
```

3. 创建一个按钮，并使其绑定一个事件方法，如：OnBnClickedListen，然后在这个方法里添加一些代码：
```c++
	// 创建一个协程用来监听指定地址，接收客户端连接请求
	m_fiberListen = new CFiberListener("127.0.0.1:8800");
	// 启动监听协程
	m_fiberListen->start();
```

4. 实现步骤 3 中指定的监听协程类
```c++
class CFiberListener : public acl::fiber
{
public:
	CFiberListener(const char* addr) : m_addr(addr) {}
private:
	~CFiberListener(void) {}
private:
	acl::string m_addr;
	acl::server_socket m_listener;
	// @override
	void run(void) {
		// 绑定并监听指定的本地地址
		if (m_listener.open(m_addr) == false) {
			return;
		}
		while (true) {
			// 等待客户端连接
			acl::socket_stream* conn = m_listener.accept();
			if (conn == NULL) {
				break;
			}
			// 创建独立的协程处理该客户端的请求
			acl::fiber* fb = new CFiberClient(conn);
			fb->start(); // 启动客户端处理协程
		}
		delete this;
	}
};
```

5. 实现步骤 4 中指定的客户端响应协程类
```c
class CFiberClient : public acl::fiber
{
public:
	CFiberClient(acl::socket_stream* conn) : m_conn(conn) {}
private:
	~CFiberClient(void) { delete m_conn; }
private:
	acl::socket_stream* m_conn;
	// @override
	void run(void) {
		char buf[8192];
		while (true) {
			// 从客户端读取数据
			int ret = m_conn->read(buf, sizeof(buf), false);
			if (ret == -1) {
				break;
			}
			// 将读到的数据回写给客户端
			if (m_conn->write(buf, ret) != ret) {
				break;
			}
		}
		delete this;
	}
};
```

通过以上步骤就可为 win32 界面程序添加基于协程模式的通信模块，上面的两个协程类的处理过程都是“死循环”的，而且又与界面过程同处同一线程运行空间，却为何却不会阻塞界面消息过程呢？其原因就是当通信协程类对象在遇到网络 IO 阻塞时，会自动将自己挂起，将线程的运行权交给其它协程或界面过程。原理就是这么简单，但内部实现还有点复杂度的，感兴趣的可以看看 Acl 协程库的实现源码(https://github.com/acl-dev/acl/tree/master/lib_fiber/ )。
此外，上面示例的完整代码请参考：https://github.com/acl-dev/acl/tree/master/lib_fiber/samples/WinEchod  。

## 十二、打印协程调用栈
当一个协程因为网络阻塞、加锁等待等原因被挂起时，有时需要打印该协程的函数调用栈以方便调试；因为协程的运行栈是我们通过分配一段动态内存模拟的运行栈，同时又因为协程又不是操作系统的调度单元，所以我们无法通过 GDB 来检查协程栈。Acl 协程目前通过结合 libunwind 库支持打印正在运行或被挂起的协程的调用栈，虽然仅支持 Linux 平台下的特定上下文切换方式（仅支持 swapcontext 上下文切换方式），但对于协程调试帮助很大。下面给出打印协程栈的过程：
- 重新设置编译条件：打开 lib_fiber/c/Makefile 文件，将上下文切换方式由 `JMP_CTX = USE_JMP_DEF` 修改为 `JMP_CTX = USE_CONTEXT`，即将协程的上下文切换方式改成 `swapcontext`; 给编译选项 `CFLAGS` 增加条件编译，即 `CFLAGS += -DDEBUG_STACK`；然后重新编译 libfiber.a 库；
- 在协程代码中通过静态方法 `acl::fiber::stacktrace` 获取指定协程的函数调用栈；
- 在可执行程序的 Makefile 里进行程序连接时，需要添加 libunwind 相关库，即：`-lunwind -lunwind-x86_64`;
- 编译可执行程序，便可以自行打印被挂起协程的调用栈了。

下面给出一个程序例子：
```c++
#include <acl-lib/acl_cpp/lib_acl.hpp>
#include <acl-lib/fiber/libfiber.hpp>

class myfiber : public acl::fiber {
public:
	myfiber(acl::fiber_tbox<bool>& box) : box_(box) {}
	~myfiber(void) {}

private:
	acl::fiber_tbox<bool>& box_;

	// @override
	void run(void) {
		func1();
	}
	void func1(void) {
		func2();
	}
	void func3(void) {
		// 等待消息，在获得消息后退出
		box_.pop();
	}
};

class checker : public acl::fiber {
public:
	checker(acl::fiber_tbox<bool>& box, acl::fiber& fb)
	: box_(box), fiber_(fb) {}
	~checker(void) {}

private:
	acl::fiber_tbox<bool>& box_;
	acl::fiber& fiber_;

	// @override
	void run(void) {
		std::vector<acl::fiber_frame> stack;
		// 获得指定协程的调用栈
		acl::fiber::stacktrace(fiber_, stack, 50);
		// 打印指定协程的函数调用栈
		for (std::vector<acl::fiber_frame>::const_iterator
			 cit = stack.begin(); cit != stack.end(); ++cit) {

			printf("0x%lx(%s)+0x%lx\r\n",
				(*cit).pc, (*cit).func.c_str(), (*cit).off);
		}

		// 通过等待消息的协程退出
		box_.push(NULL);
	}
}

int main(void) {
	acl::fiber_tbox<bool> box;
	acl::fiber* fb1 = new myfiber(box);
	fb1->start();

	acl::fiber* fb2 = new checker(box, *fb1);
	fb2->start();

	acl::fiber::schedule();

	delete fb2;
	delete fb1;

	return 0;
}
```
在上面的示例中，执行过程如下：
- 先创建 `myfiber` 协程实例，然后阻塞在 `fiber_tbox` 上等待消息通知；
- 再创建一个 `checker` 协程实例，运行后获得并打印前面所创建 `myfiber` 协程的调用栈；
- `checker` 协程最后给 `fiber_tbox` 发消息通知 `myfiber` 协程返回后退出。

编译运行上面示例，便可以得到如下结果：
```text
0x434b0d(acl_fiber_cond_timedwait)+0x22d
0x405c85(_ZN3acl10fiber_cond4waitERNS_11fiber_mutexEi)+0x25
0x403a50(_ZN3acl10fiber_tboxIbE3popEiPb)+0x9e
0x40378d(_ZN7myfiber5func3Ev)+0x35
0x403756(_ZN7myfiber5func2Ev)+0x18
0x40373c(_ZN7myfiber5func1Ev)+0x18
0x4036ff(_ZN7myfiber3runEv)+0x3f
0x427bb1(fiber_start)+0x11
0x2b7cadaad0d0(__correctly_grouped_prefixwc)+0x160
```
可以看出 myfiber 的调用栈为：  
`run->func1->func2->func3->pop->wait->acl_fiber_cond_timedwait`

结合上面输出的 pc 字段，通过工具 `addr2line` 可以获得具体源码的文件位置，当执行：  
```text
$addr2line -e fiber_stack 0x434b0d 0x405c85 0x403a50 0x40378d 0x403756 0x40373c 0x4036ff 0x427bb1
```
便得到如下信息：
```text
/home/zsx/workspace/github/acl/lib_fiber/c/src/sync/fiber_cond.c:150
/home/zsx/workspace/github/acl/lib_fiber/cpp/src/fiber_cond.cpp:22
/usr/include/acl-lib/fiber/fiber_tbox.hpp:148
/home/zsx/workspace/github/demo/c++/fiber/fiber_stack.cpp:32
/home/zsx/workspace/github/demo/c++/fiber/fiber_stack.cpp:28
/home/zsx/workspace/github/demo/c++/fiber/fiber_stack.cpp:24
/home/zsx/workspace/github/demo/c++/fiber/fiber_stack.cpp:19
/home/zsx/workspace/github/acl/lib_fiber/c/src/fiber.c:590
```

## 十三、检测协程死锁问题
在稍微复杂的协程编程中，经常会使用协程锁对共享的资源进行同步保护，如果使用的互斥锁较多，有可能会形成死锁问题，比如下面的代码：
```c++
#include <unistd.h>
#include <acl-lib/acl_cpp/lib_acl.hpp>
#include <acl-lib/fiber/libfiber.hpp>

class fiber1 : public acl::fiber {
public:
	fiber1(acl::fiber_mutex& lock1, acl::fiber_mutex& lock2)
	: lock1_(lock1), lock2_(lock2) {}
	~fiber1(void) {}

private:
	acl::fiber_mutex& lock1_;
	acl::fiber_mutex& lock2_;

	// @override
	void run(void) {
		lock1_.lock();
		sleep(1);
		lock2_.lock();
	}
};

class fiber2 : public acl::fiber {
public:
	fiber2(acl::fiber_mutex& lock1, acl::fiber_mutex& lock2)
	: lock1_(lock1), lock2_(lock2) {}
	~fiber2(void) {}

private:
	acl::fiber_mutex& lock1_;
	acl::fiber_mutex& lock2_;

	// @override
	void run(void) {
		lock2_.lock();
		sleep(1);
		lock1_.lock();
	}
};

int main(void) {
	acl::fiber_mutex lock1, lock2;

	fiber1 fb1(lock1, lock2);
	fb1.start();

	fiber2 fb2(lock1, lock2);
	fb2.start();

	acl::fiber::schedule();
	return 0;
}
```

可以看出，上面的例子中出现了死锁问题，我们似乎可以很容易看出并解决掉它，但实际应用中程序逻辑要复杂的多，有时很难找到死锁的位置及原因，给开发应用带来了很大的困扰，为此，在 Acl 协程中给出了死锁检测方法，将发生死锁的协程及锁资源都列出，方便开发者查找死锁原因：
```c++
class checker : public acl::fiber {
public:
	checker(void) {}
	~checker(void) {}

private:
	// @override
	void run(void) {
		sleep(2);
		// 打印死锁信息至标准输出
		acl::fiber_mutex::deadlock_show();
	}
};
```
我们可以创建一个独立的协程定期检测协程锁死锁状态，将这个检测协程放到上面的示例中运行，便会得到如下的死锁信息：
```text
Deadlock happened!
fiber-1:
0x42df4f(acl_fiber_mutex_lock)+0x1af
0x40415c(_ZN3acl11fiber_mutex4lockEv)+0xc
0x403314(_ZN6fiber13runEv)+0x36
0x420781(fiber_start)+0x11
0x2b6143ac30d0(__correctly_grouped_prefixwc)+0x160
Holding mutex=0x15e3040
Waiting for mutex=0x15e32c0

fiber-2:
0x42df4f(acl_fiber_mutex_lock)+0x1af
0x40415c(_ZN3acl11fiber_mutex4lockEv)+0xc
0x4033f2(_ZN6fiber23runEv)+0x36
0x420781(fiber_start)+0x11
0x2b6143ac30d0(__correctly_grouped_prefixwc)+0x160
Holding mutex=0x15e32c0
Waiting for mutex=0x15e3040
```

可以看出，协程1拥有锁的地址为 `mutex=0x15e3040`，但在等待锁 `mutex=0x15e32c0`，而协程2恰恰相反，所以这两个协程处于死锁状态。

将上面显示栈的 pc 字段使用 addr2line 便可以得到具体的位置：
```shell
$addr2line -e fiber_deadlock 0x42df4f 0x40415c 0x403314 0x420781 0x42df4f 0x40415c 0x4033f2 0x420781

/home/zsx/workspace/github/acl/lib_fiber/c/src/sync/fiber_mutex.c:494
/home/zsx/workspace/github/acl/lib_fiber/cpp/src/fiber_mutex.cpp:28
/home/zsx/workspace/github/demo/c++/fiber/fiber_deadlock.cpp:21
/home/zsx/workspace/github/acl/lib_fiber/c/src/fiber.c:590
/home/zsx/workspace/github/acl/lib_fiber/c/src/sync/fiber_mutex.c:494
/home/zsx/workspace/github/acl/lib_fiber/cpp/src/fiber_mutex.cpp:28
/home/zsx/workspace/github/demo/c++/fiber/fiber_deadlock.cpp:39
/home/zsx/workspace/github/acl/lib_fiber/c/src/fiber.c:590
```