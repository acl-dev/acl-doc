---
title: 使用 acl_cpp 库编写多线程程序
date: 2013-10-26 18:18
categories: 线程编程
---

在 《利用ACL库开发高并发半驻留式线程池程序》中介绍了如何使用 C 版本的 acl 线程库编写多线程程序，本文将会介绍如何使用 C++ 版本的 acl 线程库编写多线程程序，虽然 C++ 版 acl 线程库基于 C 版的线程库，但却提供了更为清晰简洁的接口定义（很多地方参考了 JAVA 的线程接口定义）。下面是一个简单的使用线程的例子：

```c++
#include "acl_cpp/lib_acl.hpp"

//////////////////////////////////////////////////////////////////////////

// 子线程类定义
class mythread : public acl::thread
{
public:
	mythread() {}
	~mythread() {}
protected:
	// 基类纯虚函数，当在主线程中调用线程实例的 start 函数时
	// 该虚函数将会被调用
	virtual void* run()
	{
		const char* myname = "run";
		printf("%s: thread id: %lu, %lu\r\n",
			myname, thread_id(), acl::thread::thread_self());
		return NULL;
	}
};

//////////////////////////////////////////////////////////////////////////

static void test_thread(void)
{
	const char* myname = "test_thread";
	mythread thr;  // 子线程对象实例

	// 设置线程的属性为非分离方式，以便于下面可以调用 wait
	// 等待线程结束
	thr.set_detachable(false);

	// 启动一个子线程
	if (thr.start() == false)
	{
		printf("start thread failed\r\n");
		return;
	}

	printf("%s: thread id is %lu, main thread id: %lu\r\n",
		myname, thr.thread_id(), acl::thread::thread_self());

	// 等待子线程运行结束
	if (thr.wait(NULL) == false)
		printf("wait thread failed\r\n");
	else
		printf("wait thread ok\r\n");
}

int main(void)
{
	// 初始化 acl 库
	acl::acl_cpp_init();
	test_thread();
#ifdef WIN32
	printf("enter any key to exit ...\r\n");
	getchar();
#endif
	return 0;
}
```

从上面的示例来看，使用 acl 的线程库创建使用线程还是非常简单的。打开 lib_acl_cpp/include/acl_cpp/stdlib/thread.hpp 文件，可以看到线程类的声明，其中有两个基类：acl::thread 与 acl::thread_job，在 基类 acl::thread_job 中有一个纯虚函数 run（），acl::thread 也继承自 acl::thread_job，用户的线程类需要继承 acl::thread，并且需要实现基类 acl::thread_job 的纯虚函数： run（）。当应用在主线程中调用线程实例的 start() 函数时，acl 线程库内部便创建一个子线程，子线程被创建后线程对象的 run()　函数便被调用。下面是 acl::thread 类中几个主要的方法定义：

```c++
class thread_job
{
public:
	thread_job() {}
	virtual ~thread_job() {}

	/**
	 * 纯虚函数，子类必须实现此函数，该函数在子线程中执行
	 * @return {void*} 线程退出前返回的参数
	 */
	virtual void* run() = 0;
};

class thread : public thread_job
{
public:
	thread();
	virtual ~thread();
	
	/**
	 * 开始启动线程过程，一旦该函数被调用，则会立即启动一个新的
	 * 子线程，在子线程中执行基类 thread_job::run 过程
	 * @return {bool} 是否成功创建线程
	 */
	bool start();

	/**
	 * 当创建线程时为非 detachable 状态，则可以调用此函数
	 * 等待线程结束；否则，若创建线程时为 detachable 状态
	 * 在调用本函数时将会报错
	 * @param out {void**} 当该参数非空指针时，该参数用来存放
	 *  线程退出前返回的参数
	 * @return {bool} 是否成功
	 */
	bool wait(void** out = NULL);

	/**
	 * 在调用 start 前调用此函数可以设置所创建线程是否为
	 * 分离 (detachable) 状态；如果未调用此函数，则所创建
	 * 的线程默认为分离状态
	 * @param yes {bool} 是否为分离状态
	 * @return {thread&}
	 */
	thread& set_detachable(bool yes);

	/**
	 * 在调用 start 前调用此函数可以设置所创建线程的堆栈大小
	 * @param size {size_t} 线程堆栈大小，当该值为 0 或未
	 *  调用此函数，则所创建的线程堆栈大小为系统的默认值
	 * @return {thread&}
	 */
	thread& set_stacksize(size_t size);

	/**
	 * 在调用 start 后调用此函数可以获得所创建线程的 id 号
	 * @return {unsigned long}
	 */
	unsigned long thread_id() const;

	/**
	 * 当前调用者所在线程的线程 id 号
	 * @return {unsigned long}
	 */
	static unsigned long thread_self();
        ....
};

```

从上面的线程示例及 acl::thread 的类定义，也许有人会觉得应该把 acl::thread_job 的纯虚方法：run() 放在 acl::thread 类中，甚至觉得 acl::thread_job 类是多余的，但是因为 acl 库中还支持线程池方式，则 acl::thread_job 就显得很有必要了。在 lib_acl_cpp\include\acl_cpp\stdlib\thread_pool.hpp 头文件中可以看到 acl 的线程池类 acl::thread_pool 的声明，该类的主要函数接口如下：

```c++
class thread_pool
{
	/**
	 * 启动线程池，在创建线程池对象后，必须首先调用此函数以启动线程池
	 */
	void start();

	/**
	 * 停止并销毁线程池，并释放线程池资源，调用此函数可以使所有子线程退出，
	 * 但并不释放本实例，如果该类实例是动态分配的则用户应该自释放类实例，
	 * 在调用本函数后，如果想重启线程池过程，则必须重新调用 start 过程
	 */
	void stop();

	/**
	 * 等待线程池中的所有线程池执行完所有任务
	 */
	void wait();

	/**
	 * 将一个任务交给线程池中的一个线程去执行，线程池中的
	 * 线程会执行该任务中的 run 函数
	 * @param job {thread_job*} 线程任务
	 * @return {bool} 是否成功
	 */
	bool run(thread_job* job);

	/**
	 * 将一个任务交给线程池中的一个线程去执行，线程池中的
	 * 线程会执行该任务中的 run 函数；该函数功能与 run 功能完全相同，只是为了
	 * 使 JAVA 程序员看起来更为熟悉才提供了此接口
	 * @param job {thread_job*} 线程任务
	 * @return {bool} 是否成功
	 */
	bool execute(thread_job* job);

	/**
	 * 在调用 start 前调用此函数可以设置所创建线程的堆栈大小
	 * @param size {size_t} 线程堆栈大小，当该值为 0 或未
	 *  调用此函数，则所创建的线程堆栈大小为系统的默认值
	 * @return {thread&}
	 */
	thread_pool& set_stacksize(size_t size);

	/**
	 * 设置线程池最大线程个数限制
	 * @param max {size_t} 最大线程数，如果不调用此函数，则内部缺省值为 100
	 * @return {thread_pool&}
	 */
	thread_pool& set_limit(size_t max);

	/**
	 * 设置线程池中空闲线程的超时退出时间
	 * @param ttl {int} 空闲超时时间(秒)，如果不调用此函数，则内部缺省为 0
	 * @return {thread_pool&}
	 */
	thread_pool& set_idle(int ttl);
        ......
};
```

这些接口定义也相对简单，下面给出一个使用线程池的例子：

```c++
// 线程工作类
class myjob : public acl::thread_job
{
public:
	myjob() {}
	~myjob() {}

protected:
	// 基类中的纯虚函数
	virtual void* run()
	{
		const char* myname = "run";
		printf("%s: thread id: %lu\r\n",
			myname, acl::thread::thread_self());
		return NULL;
	}
};

//////////////////////////////////////////////////////////////////////////

// 线程池类
class mythread_pool : public acl::thread_pool
{
public:
	mythread_pool() {}
	~mythread_pool()
	{
		printf("thread pool destroy now, tid: %lu\r\n",
			acl::thread::thread_self());
	}

protected:
	// 基类虚函数，当子线程被创建时该虚函数将被调用
	virtual bool thread_on_init()
	{
		const char* myname = "thread_on_init";

		printf("%s: curr tid: %lu\r\n", myname,
			acl::thread::thread_self());
		return true;
	}

	// 基类虚函数，当子线程退出前该虚函数将被调用
	virtual void thread_on_exit()
	{
		const char* myname = "thread_on_exit";

		printf("%s: curr tid: %lu\r\n", myname,
			acl::thread::thread_self());
	}
};

//////////////////////////////////////////////////////////////////////////

void test()
{
	acl::thread_pool* threads = new mythread_pool();
	threads->start();  // 启动线程池过程

	acl::thread_job *job1= new myjob, *job2 = new myjob;
	threads->execute(job1);
	threads->execute(job2);

	// 为了保证 job1, job2动态内存被正确释放，
	// 必须调用 threads->stop 等待子线程运行结束后在
	// 主线程中将其释放
	threads->stop();
	delete threads;

	// 在主线程中释放动态分配的对象
	delete job1;
	delete job2;
}
```

如上例所示，在使用 acl 的C++版本线程池类库时，必须定义一个线程工作类（继承自 acl::thread_job）并实现基类的纯虚函数：run()；另外，在使用线程池时，如果想要在线程创建时初始化一些线程局部变量以及在线程退出前释放一些线程局部变量，则可以定义 acl::thread_pool 的子类，实现基类中的 thread_on_init 和 thread_on_exit 方法，如果不需要，则可以直接使用 acl::thread_pool 类对象。

github：https://github.com/acl-dev/acl
gitee：https://gitee.com/acl-dev/acl