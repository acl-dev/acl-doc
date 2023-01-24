---
title: 编译使用Acl协程库
date: 2019-03-23 13:08:24
tags: 协程编程
categories: 协程编程
---

## 一、概述
在《使用 acl 协程编写高并发网络服务》和《使用协程方式编写高并发的 WEB 服务》两篇文章中介绍了如何使用 acl 的协程功能编写高并发服务器程序，本文主要介绍如何编译使用 acl 的网络协程库。

## 二、 acl 协程库的依赖关系
目前 acl 协程主要分为 C 库（lib_fiber.a，在 acl/lib_fiber/c 目录下）和 C++库（libfiber_cpp.a，在 acl/lib_fiber/cpp 目录下），其中 lib_fiber_cpp.a 依赖 libfiber.a，具体的依赖关系如下：
![协程库依赖](/img/fiber_depedence.png)

libfiber.a 目前是独立的库，libfiber_cpp.a 依赖 libfiber.a 和 lib_acl_cpp.a，lib_acl_cpp.a 依赖 lib_protocol.a 和 lib_acl.a，lib_protocol.a 依赖 lib_acl.a。

其中，lib_acl.a 为 acl 中的核心基础 C 库，lib_protocol.a 为 acl 中的网络协议（http/icmp/smtp）基础 C 库，lib_acl_cpp 为 C++库，依赖上述两个 C 库；libfiber.a 为独立的网络协程库，仅依赖于系统库，libfiber_cpp.a 为封装了 libfiber.a 的 C++ 库，如果用户所用的 GCC 支持 C++11，则该库还支持更为简洁的创建协程的方式（借助于 C++11中的 lambda 表达式方式）。

## 三、一个简单的例子
下面是一个简单的使用 acl 协程的例子：

```c
#include "lib_acl.h"
#include <stdio.h>
#include <stdlib.h>
#include "fiber/lib_fiber.h"

static int __max_loop = 100;
static int __max_fiber = 10;
static int __stack_size = 64000;

/* 协程处理入口函数 */
static void fiber_main(ACL_FIBER *fiber, void *ctx acl_unused)
{
    int  i;

    /* 两种方式均可以获得当前的协程号 */
    assert(acl_fiber_self() == acl_fiber_id(fiber));

    for (i = 0; i < __max_loop; i++) {
        acl_fiber_yield();  /* 主动让出 CPU 给其它协程 */
        printf("fiber-%d\r\n", acl_fiber_self());
    }
}

int main(void)
{
    int   ch, i;

    /* 创建协程 */
    for (i = 0; i < __max_fiber; i++) {
        acl_fiber_create(fiber_main, NULL, __stack_size);
    }

    printf("---- begin schedule fibers now ----\r\n");
    acl_fiber_schedule(); /* 循环调度所有协程，直至所有协程退出 */
    printf("---- all fibers exit ----\r\n");
    return 0;
}
```

上述例子非常简单，说明了 acl 协程创建、启动和运行过程，如果仅此而已，当然使用协程并没有什么卵用，协程的关键价值在于与网络通信的结合，可以达到高并发、高性能的目的。因此，现实中协程的应用范围主要还是网络服务方面，更为准确的叫法应该是“网络协程”，脱离了“网络”协程基本没啥价值。本文的开头给出了两个链接，指明了网络协程的应用场景及实例。

## 四、编译例子
下面的 Makefile 文件说明了最简单的编译方式：

```
fiber: main.o
        gcc -o fiber main.o -L./lib_fiber/lib -L./lib_acl/lib -l_acl  -lfiber -lpthread -ldl
main.o: main.c
        gcc -c main.c -O3 -DLINUX2 -I./lib_fiber/c/include -I./lib_acl/include
```
该 Makefile 也非常简单，也仅是说明了使用 acl 网络协程库时的编译条件。其中，l_fiber 指定了 acl 的网络协程库，l_acl 指定了 acl 基础库，-lpthread 及 -ldl 指定所依赖的系统库；-DLINUX2 指定 LINUX 平台的编译条件，-I 指定头文件所在位置。

## 五、C++ 示例
上面的例子展示了使用 acl C 协程库的使用方法，同时 acl 也提供了 C++ 类封装，方便 C++ 程序员编写协程应用，下面的例子为使用标准 C++ 编写的协程示例：

```c++
#include <assert.h>
#include <iostream>
#include "acl_cpp/lib_acl.hpp"
#include "fiber/lib_fiber.hpp"

class myfiber : public acl::fiber
{
public:
    myfiber(int max_loop) : max_loop_(max_loop) {}

protected:
    // @override 实现基类纯虚函数
    void run(void)
    {
        // 两种方式均可以获得当前的协程号
        assert(get_id() == acl::fiber::self());

        for (int i = 0; i < max_loop_; i++) {
            acl::fiber::yield(); // 主动让出 CPU 给其它协程
            std::cout << "fiber-" << acl::fiber::self() << std::endl;
        }

        delete this; // 因为是动态创建的，所以需自动销毁
    }

private:
    int max_loop_;

    ~myfiber(void) {}
};

int main(void)
{
    int i, max_fiber = 10, max_loop = 10;

    for (i = 0; i < max_fiber; i++) {
        acl::fiber* fb = new myfiber(max_loop); // 创建协程
        fb->start(); // 启动协程
    }

    std::cout << "---- begin schedule fibers now ----" << std::endl;
    acl::fiber::schedule(); // 循环调度所有协程，直至所有协程退出
    std::cout << "---- all fibers exit ----" << std::endl;

    return 0;
}
```

该例子也非常简单，实现了与上面 C 示例相同的功能，只是采用 C++ 而已。用户首先需要定义自己的类，其继承于 acl::fiber 协程类，然后实现协程类中的纯虚方法：run()，当调用协程的启动方法 start() 时，acl::fiber 基类会自动回调纯虚方法  run()。

该 C++ 示例的 Makefile 与 C 的有所不同，内容如下：

```
fiber: main.o
        g++ -o fiber main.o -L./lib_fiber/lib -lfiber_cpp \
                -L./lib_acl_cpp/lib -l_acl_cpp \
                -L./lib_acl/lib -l_acl -lfiber \
                -lpthread -ldl
main.o: main.cpp
        g++ -O3 -Wall -c main.cpp -DLINUX2 -I./lib_fiber/cpp/include \
                -I./lib_acl_cpp/include
```

## 六、C++11 示例
acl 的协程库同时提供了支持 C++11 方式的调用方法，使创建协程更加方便，代码如下：

```c++
#include <iostream>
#include "acl_cpp/lib_acl.hpp"
#include "fiber/lib_fiber.hpp"

static void fiber_main(int max_loop)
{
    for (int i = 0; i < max_loop; i++) {
        acl::fiber::yield(); // 主动让出 CPU 给其它协程
        std::cout << "fiber-" << acl::fiber::self() << std::endl;
    }
}

int main(void)
{
    int i, max_fiber = 10, max_loop = 10;

    for (i = 0; i < max_fiber; i++) {
        go[=] { // 采用 c++11 的 lambad 表达式方式创建协程
            fiber_main(max_loop); // 进入协程处理函数
        };
    }

    std::cout << "---- begin schedule fibers now ----" << std::endl;
    // 循环调度所有协程，直至所有协程退出
    acl::fiber::schedule();
    std::cout << "---- all fibers exit ----" << std::endl;

    return 0;
}
```

使用 C++11 方式创建协程是不是感觉更加简洁？

同样下面给出 makefile 内容：

```
fiber: main.o
        g++ -o fiber main.o -L../../../lib -lfiber_cpp \
                -L../../../../lib_acl_cpp/lib -l_acl_cpp \
                -L../../../../lib_acl/lib -l_acl  -lfiber \
                -lpthread -ldl
main.o: main.cpp
        g++ -std=c++11 -O3 -Wall -c main.cpp -DLINUX2 -I.. -I../../../cpp/include \
                -I../../../../lib_acl_cpp/include
```

## 七、基于协程的网络服务
前面提到了“如果协程不与网络应用结合，则不会发挥其价值“，因此，下面就给出一个具体的基于协程的网络服务器程序：

```c++
#include <iostream>
#include "acl_cpp/lib_acl.hpp"
#include "fiber/lib_fiber.hpp"

class fiber_client : public acl::fiber
{
public:
    fiber_client(acl::socket_stream* conn) : conn_(conn) {}

protected:
    // @override 实现基类纯虚函数
    void run(void)
    {
        std::cout << "fiber-" << acl::fiber::self()
            << ": fd=" << conn_->sock_handle()
            << ", addr=" << conn_->get_peer() << std::endl;
        echo();
        delete this; // 因为是动态创建的，所以需自动销毁
    }

private:
    acl::socket_stream* conn_;

    ~fiber_client(void)
    {
        delete conn_;
    }

    void echo(void)
    {
        char buf[8192];

        // 从客户端读取数据并回显
        while (!conn_->eof()) {
            int ret = conn_->read(buf, sizeof(buf), false);
            if (ret == -1) {
                std::cout << "read " << acl::last_serror() << std::endl;
                break;
            }
            if (conn_->write(buf, ret) == -1) {
                std::cout << "write " << acl::last_serror() << std::endl;
                break;
            }
        }
    }
};

class fiber_server : public acl::fiber
{
public:
    fiber_server(const char* addr) : addr_(addr) {}

protected:
    // @override
    void run(void)
    {
        // 监听服务地址
        acl::server_socket ss;
        if (ss.open(addr_) == false) {
            std::cout << "listen " << addr_.c_str() << " error" << std::endl;
            delete this;
            return;
        }

        std::cout << "listen " << addr_.c_str() << " ok" << std::endl;

        while (true) {
            // 等待接收客户端连接
            acl::socket_stream* conn = ss.accept();
            if (conn == NULL) {
                std::cout << "accept error" << std::endl;
                break;
            }

            // 创建客户端处理协程
            acl::fiber* fb = new fiber_client(conn);
            fb->start();
        }

        delete this;
    }

private:
    acl::string addr_;

    ~fiber_server(void) {}
};

int main(void)
{
    const char* addr = "127.0.0.1:8089";

    acl::fiber* fb = new fiber_server(addr); // 创建监听服务协程
    fb->start(); // 启动监听协程

    // 循环调度所有协程，直至所有协程退出
    acl::fiber::schedule();

    return 0;
}
```

麻雀虽小，五脏俱全，该示例简明扼要地说明了如何使用 acl 的网络协程库编写支持高并发的网络服务应用。

## 八、参考
在 acl/lib_fiber/samples/ 目录下，还有大量的使用 acl 协程的例子，包括：定时器、简单聊天服务、mysql 访问协程化、redis 访问协程化、域名解析协程化等。

github：https://github.com/acl-dev/acl
gitee: http://git.oschina.net/acl-dev/acl
