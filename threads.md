---
title: 利用ACL库开发高并发半驻留式线程池程序
date: 2009-06-07 13:41
categories: 线程编程
---

## 一、概述
在当今强调多核开发的年代，要求程序员能够写出高并发的程序，而利用多个核一般有两种方式：采用多线程方式或多进程方式。每处理一个新任务时如果临时产生一个线程或进程且处理完任务后线程或进程便立即退出，显示这种方式是非常低效的，于是人们一般采用线程池的模型（这在JAVA 或 .NET 中非常普遍）或多进程进程池模型（这一般在UNIX平台应用较多）。此外，对于线程池或进程池模型又分为两种情形：常驻留内存或半驻留内存，常驻内存是指预先产生一批线程或进程，等待新任务到达，这些线程或进程即使在空闲状态也会常驻内存；半驻留内存是指当来新任务时如果线程池或进程池没有可利用线程或进程则启动新的线程或进程来处理新任务，处理完后，线程或进程并不立即退出，而是空闲指定时间，如果在空闲阀值时间到达前有新任务到达则立即处理新任务，如果到达空闲超时后依然没有新任务到达，则这些空闲的线程或进程便退出，以让出系统资源。所以，对比常驻内存方式和半驻留内存方式，不难看出半驻留方式更有按需分配的意味。

下面仅以ACL框架中的半驻留线程池模型为例介绍了如何写一个半驻留线程池的程序。

## 二、半驻留线程池函数接口说明
### 2.1）线程池的创建、销毁及任务添加等接口
```c++
/**
 * 创建一个线程池对象
 * @param attr {acl_pthread_pool_attr_t*} 线程池创建时的属性，如果该参数为空，
 *  则采用默认参数: ACL_PTHREAD_POOL_DEF_XXX
 * @return {acl_pthread_pool_t*}, 如果不为空则表示成功，否则失败
*/
ACL_API acl_pthread_pool_t *acl_pthread_pool_create(const acl_pthread_pool_attr_t *attr);

/**
 * 销毁一个线程池对象, 成功销毁后该对象不能再用.
 * @param thr_pool {acl_pthread_pool_t*} 线程池对象，不能为空
 * @return {int} 0: 成功; != 0: 失败
 */
ACL_API int acl_pthread_pool_destroy(acl_pthread_pool_t *thr_pool);

/**
 * 向线程池添加一个任务
 * @param thr_pool {acl_pthread_pool_t*} 线程池对象，不能为空
 * @param run_fn {void (*)(*)} 当有可用工作线程时所调用的回调处理函数
 * @param run_arg {void*} 回调函数 run_fn 所需要的回调参数
 * @return {int} 0: 成功; != 0: 失败
 */
ACL_API int acl_pthread_pool_add(acl_pthread_pool_t *thr_pool,
        void (*run_fn)(void *), void *run_arg);

/**
 * 当前线程池中的线程数
 * @param thr_pool {acl_pthread_pool_t*} 线程池对象，不能为空
 * @return {int} 返回线程池中的总线程数
 */
ACL_API int acl_pthread_pool_size(acl_pthread_pool_t *thr_pool);
```

### 2.2）线程池属性设置接口
```c++
/**
 * 初始化线程池属性值
 * @param attr {acl_pthread_pool_attr_t*}
 */
ACL_API void acl_pthread_pool_attr_init(acl_pthread_pool_attr_t *attr);

/**
 * 设置线程池属性中的最大堆栈大小(字节)
 * @param attr {acl_pthread_pool_attr_t*}
 * @param size {size_t}
 */
ACL_API void acl_pthread_pool_attr_set_stacksize(acl_pthread_pool_attr_t *attr, size_t size);

/**
 * 设置线程池属性中的最大线程数限制值
 * @param attr {acl_pthread_pool_attr_t*}
 * @param threads_limit {int} 线程池中的最大线程数
 */
ACL_API void acl_pthread_pool_attr_set_threads_limit(acl_pthread_pool_attr_t *attr, 
    int threads_limit);
/**
 * 设置线程池属性中线程空闲超时值
 * @param attr {acl_pthread_pool_attr_t*}
 * @param idle_timeout {int} 线程空闲超时时间(秒)
 */
ACL_API void acl_pthread_pool_attr_set_idle_timeout(acl_pthread_pool_attr_t *attr, 
    int idle_timeout);
```

### 2.3）线程池中的工作线程创建、退出时设置回调函数接口
```c++
/**
 * 添加注册函数，在线程创建后立即执行此初始化函数
 * @param thr_pool {acl_pthread_pool_t*} 线程池对象，不能为空
 * @param init_fn {int (*)(void*)} 工作线程初始化函数, 如果该函数返回 < 0,
 *  则该线程自动退出。
 * @param init_arg {void*} init_fn 所需要的参数
 * @return {int} 0: OK; != 0: Error.
 */
ACL_API int acl_pthread_pool_atinit(acl_pthread_pool_t *thr_pool,
        int (*init_fn)(void *), void *init_arg);

/**
 * 添加注册函数，在线程退出立即执行此初函数
 * @param thr_pool {acl_pthread_pool_t*} 线程池对象，不能为空
 * @param free_fn {void (*)(void*)} 工作线程退出前必须执行的函数
 * @param free_arg {void*} free_fn 所需要的参数
 * @return {int} 0: OK; != 0: Error.
 */
ACL_API int acl_pthread_pool_atfree(acl_pthread_pool_t *thr_pool,
        void (*free_fn)(void *), void *free_arg);
```

## 三、半驻留线程池例子
### 3.1）程序代码
```c++
#include "lib_acl.h"
#include <assert.h>

/**
 * 用户自定义数据结构
 */
typedef struct THREAD_CTX {
	acl_pthread_pool_t *thr_pool;
	int   i;
} THREAD_CTX;

/* 全局性静态变量 */
static acl_pthread_pool_t *__thr_pool = NULL;

/* 线程局部存储变量(C99支持此种方式声明，方便许多) */
static __thread unsigned int __local = 0;

static void work_thread_fn(void *arg)
{
	THREAD_CTX *ctx = (THREAD_CTX*) arg; /* 获得用户自定义对象 */
	int   i = 5;

	/* 仅是验证参数传递过程 */
	assert(ctx->thr_pool == __thr_pool);

	while (i-- > 0) {
		printf("thread start! tid=%d, i=%d, __local=%d\r\n",
				acl_pthread_self(), ctx->i, __local);
		/* 在本线程中将线程局部变量加1 */
		__local++;
		sleep(1);
	}

	acl_myfree(ctx);

	/* 至此，该工作线程进入空闲状态，直到空闲超时退出 */
}

static int on_thread_init(void *arg)
{
	const char *myname = "on_thread_init";
	acl_pthread_pool_t *thr_pool = (acl_pthread_pool_t*) arg;

	/* 判断一下，仅是为了验证参数传递过程 */
	assert(thr_pool == __thr_pool);
	printf("%s: thread(%d) init now\r\n", myname, acl_pthread_self());

	/* 返回0表示继续执行该线程获得的新任务，返回-1表示停止执行该任务 */
	return (0);
}

static void on_thread_exit(void *arg)
{
	const char *myname = "on_thread_exit";
	acl_pthread_pool_t *thr_pool = (acl_pthread_pool_t*) arg;

	/* 判断一下，仅是为了验证参数传递过程 */
	assert(thr_pool == __thr_pool);
	printf("%s: thread(%d) exit now\r\n", myname, acl_pthread_self());
}

static void run_thread_pool(acl_pthread_pool_t *thr_pool)
{
	THREAD_CTX *ctx;  /* 用户自定义参数 */

	/* 设置全局静态变量 */
	__thr_pool = thr_pool;

	/* 设置线程开始时的回调函数 */
	(void) acl_pthread_pool_atinit(thr_pool, on_thread_init, thr_pool);

	/* 设置线程退出时的回调函数 */
	(void) acl_pthread_pool_atfree(thr_pool, on_thread_exit, thr_pool);

	ctx = (THREAD_CTX*) acl_mycalloc(1, sizeof(THREAD_CTX));
	assert(ctx);
	ctx->thr_pool = thr_pool;
	ctx->i = 0;

	/**
	 * 向线程池中添加第一个任务，即启动第一个工作线程
	 * @param thr_pool 线程池句柄
	 * @param workq_thread_fn 工作线程的回调函数
	 * @param ctx 用户定义参数
	 */
	acl_pthread_pool_add(thr_pool, work_thread_fn, ctx);
	sleep(1);

	ctx = (THREAD_CTX*) acl_mycalloc(1, sizeof(THREAD_CTX));
	assert(ctx);
	ctx->thr_pool = thr_pool;
	ctx->i = 1;
	/* 向线程池中添加第二个任务，即启动第二个工作线程 */
	acl_pthread_pool_add(thr_pool, work_thread_fn, ctx);
}

int main(int argc acl_unused, char *argv[] acl_unused)
{
	acl_pthread_pool_t *thr_pool;
	int  max_threads = 20;  /* 最多并发20个线程 */
	int  idle_timeout = 10; /* 每个工作线程空闲10秒后自动退出 */
	acl_pthread_pool_attr_t attr;

	acl_pthread_pool_attr_init(&attr);
	acl_pthread_pool_attr_set_threads_limit(&attr, max_threads);
	acl_pthread_pool_attr_set_idle_timeout(&attr, idle_timeout);

	/* 创建半驻留线程句柄 */
	thr_pool = acl_pthread_pool_create(&attr);
	assert(thr_pool);
	run_thread_pool(thr_pool);

	if (0) {
		/* 如果立即运行 acl_pthread_pool_destroy，则由于调用了线程池销毁函数，
		 * 主线程便立刻通知空闲线程退出，所有空闲线程不必等待空闲超时时间便可退出,
		 */
		printf("> wait all threads to be idle and free thread pool\r\n");
		/* 立即销毁线程池 */
		acl_pthread_pool_destroy(thr_pool);
	} else {
		/* 因为不立即调用 acl_pthread_pool_destroy，所有所有空闲线程都是当空闲
		 * 超时时间到达后才退出
		 */
		while (1) {
			int   ret;

			ret = acl_pthread_pool_size(thr_pool);
			if (ret == 0)
				break;
			printf("> current threads in thread pool is: %d\r\n", ret);
			sleep(1);
		}
		/* 线程池中的工作线程数为0时销毁线程池 */
		printf("> all worker thread exit now\r\n");
		acl_pthread_pool_destroy(thr_pool);
	}

	/* 主线程等待用户在终端输入任意字符后退出 */
	printf("> enter any key to exit\r\n");
	getchar();

	return (0);
}
```

### 3.2) 编译链接
从　http://www.sourceforge.net/projects/acl/ 站点下载 acl_project 代码，在WIN32平台下请用VC2003编译，打开 acl_project\win32_build\vc\acl_project_vc2003.sln 编译后在目录　acl_project\dist\lib_acl\lib\win32　下生成lib_acl_vc2003.lib, 然后在示例的控制台工程中包含该库，并包含acl_project\lib_acl\include　下的 lib_acl.h 头文件，编译上述源代码即可。
因为本例子代码在 ACL 的例子里有，所以可以直接编译 acl_project\win32_build\vc\samples\samples_vc2003.sln 中的 thread_pool 项目即可。

### 3.3) 运行
运行示例程序后，在我的机器的显示结果如下：
```
on_thread_init: thread(23012) init now
thread start! tid=23012, i=0, __local=0
thread start! tid=23012, i=0, __local=1
> current threads in thread pool is: 2
on_thread_init: thread(23516) init now
thread start! tid=23516, i=1, __local=0
thread start! tid=23516, i=1, __local=1
> current threads in thread pool is: 2
thread start! tid=23012, i=0, __local=2
thread start! tid=23516, i=1, __local=2
thread start! tid=23012, i=0, __local=3
> current threads in thread pool is: 2
thread start! tid=23516, i=1, __local=3
thread start! tid=23012, i=0, __local=4
> current threads in thread pool is: 2
thread start! tid=23516, i=1, __local=4
> current threads in thread pool is: 2
> current threads in thread pool is: 2
> current threads in thread pool is: 2
> current threads in thread pool is: 2
> current threads in thread pool is: 2
> current threads in thread pool is: 2
> current threads in thread pool is: 2
> current threads in thread pool is: 2
> current threads in thread pool is: 2
> current threads in thread pool is: 2
> current threads in thread pool is: 2
on_thread_exit: thread(23012) exit now
> current threads in thread pool is: 1
on_thread_exit: thread(23516) exit now
> all worker thread exit now
> enter any key to exit
```

## 四、小结
可以看出，使用ACL库创建半驻留式高并发多线程程序是比较简单的，ACL线程池库的接口定义及实现尽量与POSIX中规定的POSIX线程的实现接口相似，创建与使用ACL线程池库的步骤如下：
- acl_pthread_pool_attr_init: 初始化创建线程池对象所需要属性信息(可以通过 acl_pthread_pool_attr_set_threads_limit 设置线程池最大并发数及用 acl_pthread_pool_attr_set_idle_timeout 设置线程池中工作线程的空闲退出时间间隔)
- acl_pthread_pool_create: 创建线程池对象
- acl_pthread_pool_add: 向线程池中添加新任务，新任务将由线程池中的某一工作线程执行
- acl_pthread_pool_destroy: 通知并等待线程池中的工作线程执行完任务后退出，同时销毁线程池对象

还可以在选择在创建线程池对象后，调用 acl_pthread_pool_atinit 设置工作线程第一次被创建时回调用户自定义函数过程，或当线程空闲退出后调用 acl_pthread_pool_atfree 中设置的回调函数。
另外，可以将创建线程池的过程进行一抽象，写成如下形式：

```c++
/**
 * 创建半驻留线程池的过程
 * @return {acl_pthread_pool_t*} 新创建的线程池句柄
 */
static acl_pthread_pool_t *create_thread_pool(void)
{
	acl_pthread_pool_t *thr_pool;  /* 线程池句柄 */
	int  max_threads = 100;  /* 最多并发100个线程 */
	int  idle_timeout = 10;  /* 每个工作线程空闲10秒后自动退出 */
	acl_pthread_pool_attr_t attr;  /* 线程池初始化时的属性 */

	/* 初始化线程池对象属性 */
	acl_pthread_pool_attr_init(&attr);
	acl_pthread_pool_attr_set_threads_limit(&attr, max_threads);
	acl_pthread_pool_attr_set_idle_timeout(&attr, idle_timeout);

	/* 创建半驻留线程句柄 */
	thr_pool = acl_pthread_pool_create(&attr);
	assert(thr_pool);
	return (thr_pool);
}
```

其实，利用ACL创建线程池还有一个简化接口（只所以叫 acl_thread_xxx 没有加 p, 是因为这个接口不太遵守 Posix的一些规范），如下：

```c++
/**
 * 更简单地创建线程对象的方法
 * @param threads_limit {int}  线程池中最大并发线程数
 * @param idle_timeout {int} 工作线程空闲超时退出时间(秒)，如果为0则工作线程永不退出
 * @return {acl_pthread_pool_t*}, 如果不为空则表示成功，否则失败
 */
ACL_API acl_pthread_pool_t *acl_thread_pool_create(int threads_limit, int idle_timeout);
```

这样，用户就可以非常方便地创建自己的线程池了，而且别忘了，这个线程池还是可以是半驻留的（当然也是跨平台的，可以运行在　Linux/Solaris/FreeBSD/Win32 的环境下）。

github：https://github.com/acl-dev/acl/
gitee：https://gitee.com/acl-dev/acl/