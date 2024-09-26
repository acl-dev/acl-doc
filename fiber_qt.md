---
title: 在 QT 界面编程中使用协程
date: 2024-09-24 21:01:24
tags: 协程编程
categories: 协程编程
---

# 一、概述

人们在谈论协程编程时，往往与编写命令行网络程序有关，如编写网络客户端与网络服务器程序，很少涉及到客户端 UI 相关的界面编程。Acl 协程库是支持在 Windows 下的 UI 界面编程的，因为 [Acl](https://github.com/acl-dev/acl/) 协程的事件引擎支持了界面消息传递过程。最近学习了一下 QT UI 编程，轻松将 Acl 协程与 QT UI 集成在一起，从而实现了 QT 界面协程化，使开发人员在使用 QT 编写界面程序时，编写网络模块变得非常简单。

本文结合 Acl 中 lib_fiber/samples-gui/QtFiber 示例，演示了如何将 Acl 协程功能集成到 QT 界面中，实现了网络模块与界面模块的融合。

# 二、集成
## 2.1、编译 Acl

目前 QT IDE 还无法直接使用 Acl 里的 CMakeLists.txt 文件编译 ACL，可以借助于 VC2019 打开 Acl 里的 acl_cpp_vc2019.sln 工程编译 Acl 五个库的动态库，分别为：lib_acl.dll, lib_protocol.dll, lib_acl_cpp.dll, libfiber.dll, libfiber_cpp.dll 及静态导出库：lib_acl.lib, lib_protocol.lib lib_acl_cpp.lib, libfiber.lib, libfiber_cpp.lib。

## 2.2、将 Acl 库集成到 QT 项目中

参考 lib_fiber/samples-gui/QtFiber/CMakeLists.txt 文件，将 Acl 库的头文件包含进去，如下：
```txt
set(acl_path ../../..)

include_directories(
    ${acl_path}/lib_acl/include
    ${acl_path}/lib_acl_cpp/include
    ${acl_path}/lib_fiber/c/include
    ${acl_path}/lib_fiber/cpp/include
)
```

然后设定编译条件：
```txt
add_definitions("-DACL_DLL"
    "-DACL_CPP_DLL"
    "-DHTTP_DLL"
    "-DICMP_DLL"
    "-DSMTP_DLL"
    "-DFIBER_CPP_DLL"
    "-D_CRT_SECURE_NO_WARNINGS"
    "-D_WINSOCK_DEPRECATED_NO_WARNINGS"
)
```

添加库到工程中，如下：
```txt
if (CMAKE_BUILD_TYPE STREQUAL "RELEASE")
    set(acl_libs_path ${CMAKE_CURRENT_SOURCE_DIR}/../../../x64/ReleaseDll)
else()
    set(acl_libs_path ${CMAKE_CURRENT_SOURCE_DIR}/../../../x64/DebugDll)
endif()

set(lib_all ${acl_libs_path}/libfiber_cpp.lib
    ${acl_libs_path}/lib_acl_cpp.lib
    ${acl_libs_path}/lib_protocol.lib
    ${acl_libs_path}/lib_acl.lib
    ${acl_libs_path}/libfiber.lib)

target_link_libraries(QtFiber PRIVATE Qt5::Widgets ${lib_all} Ws2_32)

add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
        "${acl_libs_path}/libfiber_cpp.dll"
        "${acl_libs_path}/libfiber.dll"
        "${acl_libs_path}/lib_acl_cpp.dll"
        "${acl_libs_path}/lib_acl.dll"
        "${acl_libs_path}/lib_protocol.dll"
        $<TARGET_FILE_DIR:${PROJECT_NAME}>
)
```

## 2.3、开始编写代码

经过摸索研究，想要集成 Acl 协程到 QT UI 程序中，需要采用以下方法（主要是协程的初始化及退出）：

### 2.3.1、QT 程序初始化时初始化 Acl 协程

在调用 QT APP exec() 前，需要先调用 Acl 协程初始化过程，如下：
```c++
static void startupCallback() {
    acl::fiber::schedule_gui(); // Won't return until schedule finished.
}

void main() {
    QApplication app(argc, argv);

    MainWindow w;
    w.show();

    QTimer::singleShot(0, startupCallback);

    app.exec();
}
```

可以看出，在调用 app.exec() 前注入了启动函数 startupCallback()，在里面启动了 acl 在界面模式下的协程调度过程 acl::fiber::schedule_gui()，该方法将进入界面消息循环过程，直到协程调度停止后才会返回。

### 2.3.2、在界面中创建协程

一旦协程调度器启动，就可以创建并运行协程了，可以在主界面上添加一个按钮，当点击该按钮后的处理函数中便可以创建并启动一个协程。比如在例子中，点击 "Start fiber server" 按钮，在处理函数 ` MainWindow::onStartServer()` 中，可以创建一个网络监听服务器，如下：
```c++
void MainWindow::onStartServer() {
    ...
    server_ = new fiber_server("127.0.0.1", 9001, this);
    server_->start();
    ...
}
```
这样在界面里就创建了一个 TCP 监听协程，当有连接连接监听地址时，在监听协程里便可以创建一个客户端连接处理协程进行处理，如下：
```c++
    while (true) {
        SOCKET conn = socket_accept(sock);
        if (conn == INVALID_SOCKET) {
            break;
        }

        acl::fiber* fb = new fiber_echo(conn);
        fb->start();
    }
```

上面例子的客户端协程启动后，便可以进行网络 IO 读写，如下：

```c++
    char buf[8192];
    while (true) {
        int ret = acl_fiber_recv(conn_, buf, sizeof(buf) - 1, 0);
        if (ret == -1) {
            break;
        }

        buf[ret] = 0;
        if (acl_fiber_send(conn_, buf, ret, 0) != ret) {
            break;
        }
    }
```

### 2.3.3、界面程序退出前需要停止协程调度

必须保证在界面程序退出前停止协程调度器，否则界面程序无法正常退出，该步骤也非常重要。可以在主界面处理类里重载基类的 ` void closeEvent(QCloseEvent *event);` 方法，在该方法里停止协程调度器，如下：
```c++
void MainWindow::closeEvent(QCloseEvent *event) {
    acl::fiber::schedule_stop(); // 停止协程调度器
    event->accept();             // 接受关闭事件
}
```

### 2.3.4、在界面线程中下载数据

点击主界面中点击HTTP下载按钮，在事件处理函数中创建协程从后端HTTP服务器下载数据，过程如下：
```c++
void MainWindow::onUrlGet() {
    ...
    go[this] {
        const char *url = "http://www.baidu.com/";
        acl::http_request req(url);
        if (!req.request(nullptr, 0)) {
            printf("Send HTTP request failed\r\n");
            return;
        }
        acl::string body;
        if (!req.get_body(body)) {
            printf("Get HTTP body error\r\n");
            return;
        }
        qDebug() << "Got body:" << body.c_str();
        ...
    };
}
```

### 2.3.5、在协程中延迟创建容器

如果想某个容器延迟创建，不必借助定时器，直接在协程中就可以轻松实现：
```c++
void MainWindow::delayCreate() {
    go[this] {
        acl::fiber::delay(5000); // 休眠 5 秒
        InputDialog dialog(this);
        dialog.exec();
    };
    qDebug() << "Fiber was created to create one window after a while";
}
```

## 2.4、效果展示

编译运行 acl/lib_fiber/samples-gui/QtFiber/ 工程，可以得到以下运行界面：
![fiber_qt](/img/fiber_qt.jpg)
- 在前面窗口中，右边请求HTTP服务器时的HTTP请求头，右连接为后端服务器返回的HTTP响应头，该下载过程中在协程中进行，运行结果显示在主界面上；
- 窗口下方的进度条为客户端协程与服务端协程交互时的交互进度展示。

## 2.5、小结

以上便是如何编译集成 Acl 协程到 QT 界面程序的方法，主要的要点是：
- 需要使用 vc2019 编译 Acl 的动态库，并集成至 QT 界面程序的工程文件中；
- 编程时需要注意两点：
  - 在启动 QT （即调用 app.exec()）前，需要先启动 Acl 协程调度器；
  - 在主界面类里需要重载基类关闭虚方法 `closeEvent()`，并在该方法里停止 Acl 协程调度器。

**注：** Acl库下载：https://github.com/acl-dev/acl/