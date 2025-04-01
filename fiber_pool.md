---
title: Acl 协程池使用指南
date: 2025-03-31 21:00:00
tags: 协程编程
categories: 协程编程
---

# Acl 协程池使用指南

## 一、说明

相对于线程而言，协程更加轻量，占用更少的资源，但协程也是有成本的，如果频繁地创建与释放协程，也会占用大量的CPU，而使用协程池便可以有效地解决这一问题；Acl协程模块提供了协程池功能，采用C++11语法，简单易用，下面是Acl协程池的功能要点：
- **动态性：** 这就意味着池子的大小在最小值与最大值之间浮动，按需创建与使用；
- **半驻留：** 协程池中的协程空闲一段时间后，如果没有新的待执行任务且池子中协程数量大于最小值时，会自动退出；
- **同步异步：** 通过调整任务缓冲大小，实现任务以同步或异步执行，同步模式使任务执行更具实时性，而异步模式则可以提升并行执行效率；
- **共享栈：** 可以使协程池中的协程以“共享栈”模式运行，当池子中数量非常多时，可以有效地减少内存占用。

## 二、使用举例

### 2.1 一个简单的示例

```c++
void mytest(int i) {
    printf("Task %d is running\n", i);
}

void test() {
    // 创建一个协程对象，各个参数如下：
    // 最小协程数：1
    // 最大协程数：20
    // 协程空闲退出时间：60000毫秒
    // 协程池中任务队列的缓冲区大小：500
    // 每个协程栈大小：64000字节
    // 协程池中的协程是否采用共享栈模式：否
    acl::fiber_pool pool(1, 20, 60000, 500, 64000, false);
    int i = 0;

    // 在协程池中执行第一个任务，捕获外部变量.
    pool.exec([i]() {
        printf("Task %d is running\n", i);
    });
    i++;

    // 在协程池中执行第二个任务，通过变参方式传递参数.
    pool.exec([](int i) {
  	  printf("Task %d is running\n", i);
    }, i);
    i++;

    // 在协程池中执行第三个任务，将变参传递给普通函数.
    pool.exec(mytest, i);

    // 开始协程调度过程.
    acl::fiber::schedule();
}
```

上面例子展示了从协程池的创建到任务添加与执行的过程，期中任务添加方法`exec`在添加任务时，允许用户使用lambda表达式或普通函数做为执行过程，并允许捕获外部变量及传递变参。下面给出协程池创建时各个参数的含义：

- **最大最小协程数：** 这两个参数分别限制了协程池中所创建协程的最大最小数量，当要执行的任务比较多且繁重时，所创建的协程可能就比较多，否则创建的协程就比较少；
- **空闲协程生命周期：** 当任务比较少时，如果总协程数大于`最小协程数`，则有些空闲的协程就会自动退出，以减少资源使用；
- **任务池缓冲：** 在创建协程池时，任务缓冲大小表明了任务添加的异步性，即执行`exec`时任务的添加与执行是异步的，不需要等待一个协程收到任务后该方法才返回；如果想要同步添加任务，则可以将任务缓冲buf设为0；
- **协程栈大小及共享栈：** 协程池中每个所建协程的栈空间大小是可以单独设定的，另外，还可以指定这些协程是否采用`共享协程栈`方式；当协程数量比较多时，采用`共享协程栈`方式可以节省较多内存。

### 2.2 等待任务完成示例

如果想要执行一批任务，并且等待所有任务完成，该如何实现？下面稍微修改前面例子，增加同步机制来等待一批任务的完成。

```c++
void mytest(acl::wait_group& wg, int i) {
    printf("Task %d is running\n", i);
    wg.done();  // 通知完成当前任务
}

void test() {
    acl::fiber_pool pool(1, 20, 60, 500, 64000, false);
    acl::wait_group wg;
    int i = 0;

    wg.add(1);
    pool.exec([&wg, i]() {
        printf("Task %d is running\n", i);
        wg.done();  // 通知完成当前任务
    });
    i++;

    wg.add(1);
    pool.exec([&wg](int i) {
  	    printf("Task %d is running\n", i);
        wg.done();  // 通知完成当前任务
    }, i);
    i++;

    wg.add(1);
    pool.exec(mytest, std::ref(wg), i);

    go[&wg, &pool] {  // 创建独立的协程等待所有任务完成
        wg.wait();    // 等待所有任务完成
        printf("All tasks finished!\r\n");
        pool.stop();  // 停止协程池
    };
}
```

该例子通过引入 acl::wait_group 对象，实现了任务完成后的同步过程。

## 三、参考链接
- **一个协程池例子：** https://github.com/acl-dev/demo/blob/master/c%2B%2B1x/fiber/fiber_pool.cpp
- **非阻塞IO+协程池回显例子：** https://github.com/acl-dev/demo/blob/master/c%2B%2B1x/fiber/fiber_nio_echod.cpp
- **非阻塞IO+协程池HTTP服务：** https://github.com/acl-dev/demo/blob/master/c%2B%2B1x/fiber/fiber_nio_httpd.cpp
- **下载Acl工程：** https://github.com/acl-dev/acl/
- **Acl 网络协程框架编程指南：** https://acl-dev.cn/2019/04/07/fiber/
