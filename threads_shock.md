---
title: 线程池设计中的惊群问题
date: 2014-03-09 00:30
categories: 线程编程
---

多线程编程已经是现在网络编程中常用的编程技术，设计一个良好的线程池库显得尤为重要。在 UNIX（WIN32下可以采用类似的方法，acl 库中的线程池是跨平台的） 环境下设计线程池库主要是如何用好如下系统 API：

- 1、pthread_cond_signal/pthread_cond_broadcast：生产者线程通知线程池中的某个或一些消费者线程池，接收处理任务；
- 2、pthread_cond_wait：线程池中的消费者线程等待线程条件变量被通知；
- 3、pthread_mutex_lock/pthread_mutex_unlock：线程互斥锁的加锁及解锁函数。

下面的代码示例是大家常见的线程池的设计方式：

```c
// 线程任务类型定义
struct thread_job {
	struct thread_job *next;  // 指向下一个线程任务
	void (*func)(void*);      // 应用回调处理函数 
	void *arg;                // 回调函数的参数
	...
};

// 线程池类型定义
struct thread_pool {
	int   max_threads;        // 线程池中最大线程数限制
	int   curr_threads;       // 当前线程池中总的线程数
	int   idle_threads;       // 当前线程池中空闲的线程数
	pthread_mutex_t mutex;    // 线程互斥锁
	pthread_cond_t  cond;     // 线程条件变量
	thread_job *first;        // 线程任务链表的表头
	thread_job *last;         // 线程任务链表的表尾
	...	
}

// 线程池中的消费者线程处理过程
static void *consumer_thread(void *arg)
{
	struct thread_pool *pool = (struct thread_pool*) arg;
	struct thread_job  *job;
	int   status;

	// 该消费者线程需要先加锁
	pthread_mutex_lock(&pool->mutex);

	while (1) {
		if (pool->first != NULL) {
			// 有线程任务时，则取出并在下面进行处理
			job = pool->first;
			pool->first = job->next;
			if (pool->last == job)
				pool->last = NULL;

			// 解锁，允许其它消费者线程加锁或生产者线程添加新的任务
			pthread_mutex_unlock(&pool->mutex);

			// 回调应用的处理函数
			job->func(job->arg);

			// 释放动态分配的内存
			free(job);

			// 重新去加锁
			pthread_mutex_lock(&pool->mutex);
		} else {
			pool->idle_threads++;

			// 在调用 pthread_cond_wait 等待线程条件变量被通知且自动解锁
			status = pthread_cond_wait(&pool->cond, &pool->mutex);

			pool->idle_threads--;

			if (status == 0)
				continue;

			// 等待线程条件变量异常，则该线程需要退出
			pool->curr_threads--;
			pthread_mutex_unlock(&pool->mutex);
			break;
		}
	}

	return NULL;
}

// 生产者线程调用此函数添加新的处理任务
void add_thread_job(struct thread_pool *pool, void (*func)(void*), void *arg)
{
	// 动态分配任务对象
	struct thread_job *job = (struct thread_job*) calloc(1, sizeof(*job));

	job->func = func;
	job->arg = arg;

	pthread_mutex_lock(&pool->mutex);

	// 将新任务添加进线程池的任务链表中
	if (pool->first == NULL)
		pool->first = job;
	else
		pool->last->next = job;
	pool->last = job;
	job->next = NULL;
	
	if (pool->idle_threads > 0) {
		// 如果有空闲消费者线程，则通知空闲线程进行处理，同时需要解锁

		pthread_mutex_unlock(&pool->mutex);
		pthread_cond_signal(&pool->cond);
	} else if (pool->curr_threads < pool->max_threads) {
		// 如果未超过最大线程数限制，则创建一个新的消费者线程

		pthread_t id;
		pthread_attr_t attr;

		pthread_attr_init(&attr);

		// 将线程属性设为分离模式，这样当线程退出时其资源自动由系统回收
		pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);

		// 创建一个消费者线程
		if (pthread_create(&id, &attr, consumer_thread, pool) == 0)
			pool->curr_threads++;

		pthread_mutex_unlock(&pool->mutex);
		pthread_attr_destroy(&attr);
	}
}

// 创建线程池对象
struct thread_pool *create_thread_pool(int max_threads)
{
	struct thread_pool *pool = (struct thread_pool*) calloc(1, sizeof(*pool));
	
	pool->max_threads = max_threads;
	pthread_mutex_init(&pool->mutex);
	pthread_cond_init(&pool->cond);
	...

	return pool;
}

///////////////////////////////////////////////////////////////////////////////////
// 使用上面线程池的示例如下：

// 由消费者线程回调的处理过程
static void thread_callback(void* arg)
{
      ...
}

void test(void)
{
	struct thread_pool *pool = create_thread_pool(100);
	int   i;

	// 循环添加 1000000 次线程处理任务
	for (i = 0; i < 1000000; i++)
		add_thread_job(pool, thread_callback, NULL);
}
```

乍一看去，似乎也没有什么问题，象很多经典的开源代码中也是这样设计的，但有一个重要问题被忽视了：线程池设计中的惊群现象。大家可以看到，整个线程池只有一个线程条件变量和线程互斥锁，生产者线程和消费者线程（即线程池中的子线程）正是通过这两个变量进行同步的。生产者线程每添加一个新任务，都会调用 pthread_cond_signal 一次，由操作系统唤醒一个在线程条件变量等待的消费者线程，但如果查看 pthread_cond_signal API 的系统帮助，你会发现其中有一句话：调用此函数后，系统会唤醒在相同条件变量上等待的一个或多个线程。而正是这句模棱两可的话没有引起很多线程池设计者的注意，这也是整个线程池中消费者线程收到信号通知后产生惊群现象的根源所在，并且是消费者线程数量越多，惊群现象越严重----意味着 CPU 占用越高，线程池的调度性能越低。

要想避免如上线程池设计中的惊群问题，在仍然共用一个线程互斥锁的条件下，给每一个消费者线程创建一个线程条件变量，生产者线程在添加任务时，找到空闲的消费者线程，将任务置入该消费者的任务队列中同时只通知 (pthread_cond_signal) 该消费者的线程条件变量，消费者线程与生产者线程虽然共用相同的线程互斥锁（因为有全局资源及调用 pthread_cond_wait 所需），但线程条件变量的通知过程却是定向通知的，未被通知的消费者线程不会被唤醒，这样惊群现象也就不会产生了。

当然，还有一些设计上的细节需要注意，比如：当没有空闲消费者线程时，需要将任务添加进线程池的全局任务队列中，消费者线程处理完自己的任务后需要查看一下线程池中的全局任务队列中是否还有未处理的任务。

更多的线程池的设计细节请参考 acl (https://sourceforge.net/projects/acl/) 库中 lib_acl/src/thread/acl_pthread_pool.c 中的代码。

github：https://github.com/acl-dev/acl
gitee： https://github.com/acl-dev/acl