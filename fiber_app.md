# ACL协程库场景化编程

## 1、概述

ACL协程库是一个用于创建协程的库，它可以帮助开发者编写高效的并发代码。ACL协程库的优势包括：

- 轻量级：ACL协程库的代码量非常小，可以轻松地集成到现有的项目中。
- 高效性：ACL协程库使用了非常高效的协程调度算法，可以在不同的操作系统和硬件平台上运行。
- 易用性：ACL协程库提供了简单易用的API，可以帮助开发者快速创建协程。

本文将实践应用场景出发，一步步介绍如何使用ACL协程库中的不同功能模块。

## 2、编写第一个协程服务器

下面是一个简单的支持回显功能的协程服务器：
```c++
#include <stdio.h>
#include <stdlib.h>
#include <acl-lib/acl_cpp/lib_acl.hpp>  // Acl c++ 基础库
#include <acl-lib/fiber/libfiber.hpp>   // Acl c++ 协程库

class fiber_client : public acl::fiber {
public:
    fiber_client(acl::socket_stream* conn) : conn_(conn) {}

private:
    acl::socket_stream* conn_;
    ~fiber_client() { delete conn_; }

protected:
    // @override，重载基类中的纯虚方法
    void run() {
        char buf[8192];

        while (true) {
            // 等待接收客户端数据
            int ret = conn_->read(buf, sizeof(buf), false);
            if (ret == -1) {
                printf("Client disconnected!\r\n");
                break;
            }

            // 回显客户端发来的数据
            if (conn_->write(buf, (size_t) ret) == -1) {
                printf("Write to client error(%s)\r\n", acl::last_serror());
                break;
            }
        }
        delete this;
    }
};

class fiber_server : public acl::fiber {
public:
    fiber_server(acl::server_socket& server) : server_(server) {}

private:
    acl::server_socket& server_;
    ~fiber_server() {}

protected:
    // @override，重载基类中的纯虚方法
    void run() {
        while (true) {
            // 等待客户端连接
            acl::socket_stream* conn = server_.accept();
            if (conn == NULL) {
                printf("Accept error(%s)\r\n", acl::last_serror());
                break;
            }

            // 创建并启动一个客户端协程，将接收到的连接对象与其绑定
            acl::fiber* fb = new fiber_client(conn);
            fb->start();
        }
        delete this;
    }
};

int main() {
    const char* addr = "127.0.0.1:8088";
    acl::server_socket server;
    if (!server.open(addr)) {  // 绑定并监听服务端口
        printf("Listen %s error(%s)\r\n", addr, acl::last_serror());
        return 1;
    }

    printf("Listen on %s ok\r\n", addr);

    // 创建并启动一个服务器协程
    acl::fiber* fb = new fiber_server(server);
    fb->start();

    // 启动协程调度器
    acl::fiber::schedule();
    return 0;
}
```
可以看出，不足百行便可以轻松写出一个支持TCP回显功能的基于协程的服务器程序（虽然这个程序目前还没什么实际用处）。在这个示例中只需注意以下几点：
- 每个应用协程类在继承底层基类 `acl::fiber` 时，需要重载并实现 `run()` 纯虚方法；
- 通过调用 `acl::fiber::schedule()` 启动协程调度器后，协程才会真正运行。

## 3、使用c++11简化协程编写

虽然上面的协程服务器已经非常简单，但我们还可以写的更简洁些，使用 c++11 中的一些语法可以简化为：

```c++
#include <stdio.h>
#include <stdlib.h>
#include <acl-lib/acl_cpp/lib_acl.hpp> // Acl 基础库
#include <acl-lib/fiber/go_fiber.hpp>  // go lamada 表达式

int main() {
    const char* addr = "127.0.0.1:8088";
    acl::server_socket server;
    if (!server.open(addr)) {
        printf("Listen on %s error(%s)\r\n", addr, acl::last_serror());
        return 1;
    }

    go[&server] { // 创建协程用来接收客户端连接
        while (true) {
            // 使用 std::shared_ptr 管理 acl::socket_stream 对象
            acl::shared_stream conn = server.shared_accept();
            if (conn == nullptr) {
                break;
            }
            go[conn] {  // 创建协程用来处理客户端数据读写
                char buf[8192];
                while (true) {
                    int ret = conn->read(buf, sizeof(buf), false);
                    if (ret == -1) {
                        printf("Client disconnected\r\n");
                        break;
                    }
                    if (conn->write(buf, ret) == -1) {
                        printf("Write to client error(%s)\r\n", acl::last_serror());
                        break;
                    }
                }
            };
        }
    };

    acl::fiber::schedule();
    return 0;
}
```
借助于c++11强大的表达能力，前面用C++98编写的协程服务器示例使用c++11改写后，代码行数减少一半，不足50行，看上去简洁许多。

## 4、指定调度器事件引擎及协程栈大小

在前面的例子中，并未指定协程调度器所用的事件引擎，也未设定所创建协程时栈大小，虽然默认值大多数情况下已经满足要求，但在一些特定场景，需要我们手工设置这些参数。

### 4.1、设定调度器事件引擎

Acl 协程调度器支持当前各主流操作系统的事件引擎：select/poll/epoll/ioruing/kqueue/iocp/Win GUI，在启动协程调度器时，API `acl::fiber::schedule(acl::fiber_event_t type)` 中的 type 为事件引擎，默认值为 `acl::FIBER_EVENT_T_KERNEL`，即使用当前系统平台的高效 Kernel 级别的事件引擎，下面给出了不同的事件引擎类型及所支持的系统：
- **acl::FIBER_EVENT_T_KERNEL：** epoll(Linux)，kquque(FreeBSD)，iocp(Windows)
- **acl::FIBER_EVENT_T_POLL：** Linux，FreeBSD，MacOS，Windows
- **acl::FIBER_EVENT_T_SELECT：** Linux，FreeBSD，MacOS，Windows
- **acl::FIBER_EVENT_T_WMSG：** Windows
- **acl::FIBER_EVENT_T_IO_URING：** Linux

如果想要在 Windows 平台上使 Windows 界面中的通信过程支持协程方式，则启动协程时可以：
```c++
acl::fiber::schedule(acl::FIBER_EVENT_T_WMSG);
```
### 4.2、指定协程运行栈大小

因为 Acl 协程是有栈协程，每个协程栈都有大小限制，为了在有限内存条件下创建更多的协程，一般会设置较小的协程栈（比如默认值为 320KB），但如果协程栈过小或程序运行时局部栈变量过大或递归层级过多，均会导致运行中的协程因栈空间不足而造成栈溢出的崩溃。下面给出了修改协程栈的方法：
```c++
size_t stack_size = 512000;
acl::fiber* fb = new myfiber(...);
fb->start(stack_size);  // 创建协程的栈空间最大为512KB
```

### 4.3、协程共享栈

在上面的例子中，如果将每个协程栈的大小设为 512KB（且协程运行过程中栈大小最多也可达到512KB），则如果服务器有 10GB 内存，最多也只能创建 20000 协程，这变得非常不划算（因为我们还期望采用协程方式支持更大并发）。如果仔细协程在运行过程中的状态切换可以看出，一般协程上下文发生切换的时机是因为 IO 过程发生了阻塞，从而导致协程被挂起，此时的协程栈大小大部分情况下占用并不多（即协程被挂起时的时间点往往不是协程占用栈空间最多的时刻），因此我们采用一种被称为“拷贝共享栈”的方式来优化内存的使用：
- 线程中的所有协程的运行栈只有一个，当任何时刻，只有一协程的栈运行在该运行栈上；
- 当处于运行状态的协程因 IO 等原因需要被挂起时，则将运行栈上数据拷贝到该协程的堆空间中；
- 当某个挂起协程被唤醒时，会把其堆上存放的程序运行数据（包括 SP、IP 等指令以及程序运行时保留的局部变量空间）拷贝到公共运行栈上。

注意：共享栈也非共享栈的区别在于拷贝内容不同，对于非共享栈，每次挂起、唤醒拷贝保留的内容主要为 SP、IP等指令指针，拷贝的内容是较少的；而对于共享栈方式，则不仅要拷贝 SP、IP 指令指针，而且还要拷贝程序运行过程被压栈的一些临时变量等，所以要拷贝的内容更多。下面是启动“共享栈”协程的方式：

```c++
size_t stack_size = 16000;
bool share_stack = true;
acl::fiber* fb = new myfiber(...);
fb->start(stack_size, share_stack);
```
其中，stack_size 参数为协程初始的堆空间大小，用来存放协程在挂起时的临时数据；share_stack 为 true 表示采用共享栈运行方式；注意，stack_size 在“共享栈”和“非共享栈”两种方式下的含义有所不同，在非共享栈方式下，该大小限制了协程的最大栈大小，在共享栈方式，该值仅为在协程挂起时需要拷贝数据的大小限制（如果实际要拷贝的数据比该值大，Acl 协程内部会自动增加空间）.

在共享栈方式下，因为只有一个运行栈，所以可以把这个共享运行栈的空间设的大些，内部缺省值为 10MB，该值也可以通过 `acl::set_shared_stack_size()` API 在进程初始化时进行设置。共享栈运行方式下，可以使我们创建更多的协程用来处理客户端连接。但共享栈方式也存在以下两个缺点：
- 在协程切换进行数据拷贝会影响切换性能（相对于非共享栈时仅拷贝几个指令而言）；
- 创建在栈空间上的对象不能在不同协程之间共享，因为协程一旦被挂起，其创建在栈上对象的指针所占空间已经属于其它协程，举个例子如下：

```c++
class fiber2 : public acl::fiber {
public:
    fiber2(int& n) : n_(n) {}

private:
    ~fiber2() {}

protected:
    // @override
    void run() {
        n_++;   // 因为 n_ 是在共享栈协程 fiber1 的栈上创建的，当运行至此时，
                // 其地址空间已变，所以此处将因非法内存访问而出现异常.
        delete this;
    }

private:
    int& n_;
};

class fiber1 : public acl::fiber {
public:
    fiber1() {}

private:
    ~fiber1() {}

protected:
    // @override
    void run() {
        int n = 100;
        acl::fiber* fb = new fiber2(n);
        fb->start();
        acl::fiber::delay(100);
        printf("n is %d\r\n", n);

        delete this;
    }
};

void test() {
    acl::fiber* fb = new fiber1;
    fb->start(16000, true);  // 第二个参数表示该协程运行在共享栈上
    acl::fiber::schedule();
}
```

上面创建的协程 fiber1 是运行在共享栈上的，在 fiber1 运行时创建一个非共享栈协程 fiber2，同时将创建于自身栈上的对象 n 传入 fiber2 中，在协程 fiber2 运行时，对创建于 fiber1 栈上的局部对象 n 进行操作，此时，n 所属的地址空间已经被 fiber2 栈所占用，而 n 原来是指向 fiber1 上的局部栈空间中，所以在 fiber2 中操作 n 时便产生了内存非法访问，最终造成内存访问异常。

## 5、如何停止正处于挂起状态的协议

如果你基于协程方式设计了一个聊天服务程序，给每个客户连接分配两个协程：一个读协程和一个写协程，其中，读协程因阻塞在读客户端数据而挂起，写协程因为等待发送消息而挂起。一个用户用某个账户名登入后，绑定在该用户连接上的两个协程均处于挂起状态，此时，该用户用相同的账户名在其它设备登入，需要将前一个“自己”踢除。为此，在 Acl 协程中提供了 `acl::fiber:kill()` API 用来将其它协程唤醒，被唤醒协程可以通过调用 `acl::fiber::self_killed()` 来判断是否被其它协程唤醒。下面给出一个例子：
```c++
class fiber_killee : public acl::fiber {
public:
    fiber_killee() {}

private:
    ~fiber_killee() {}

protected:
    // @override
    void run() {
        while (true) {
            acl::fiber::delay(1000); // 因休眠而挂起
            if (acl::fiber::self_killed()) {  // 被其它协程通知退出
                printf("The fiber has been killed!\r\n");
                break;
            }
        }
        delete this;
    }
};

class fiber_killer : public acl::fiber {
public:
    fiber_killer(acl::fiber* fb) : fb_(fb) {}

private:
    acl::fiber* fb_;
    ~fiber_killer() {}

protected:
    // @override
    void run() {
        acl::fiber::delay(1000);
        fb_->kill();  // 通知协程退出
        delete this;
    }
};

int main() {
    acl::fiber* killee = new fiber_killee;
    killee->start();

    acl::fiber* killer = new fiber_killer(killee);
    killer->start();

    acl::fiber::schedule();
    return 0;
}
```