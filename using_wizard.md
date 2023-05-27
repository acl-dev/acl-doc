---
title: 使用向导快速生成服务程序
date: 2023-05-27 21:57:00
tags: 服务编程
categories: 服务编程
---

# 使用向导快速生成服务程序
## 一、概述
在 Acl 工程提供了一个简单实用的工具（位于：acl/app/wizard 目录下），可协助使用者快速生成基于 Acl 库的服务器程序，所生成的服务器程序包含了 Acl 服务器编程框架中的多种服务模型（进程池模型、线程池模型、非阻塞模型、协程模型、触发器模型），同时面向 HTTP 开发者提供了基于进程池模型、线程池模型及协程模型的 HTTP 服务程序的生成模块。本文以生成一个基于协程的 HTTP 服务器为例介绍向导程序的使用过程。

## 二、使用向导工具

### 2.1、下载与编译
- 从 https://github.com/acl-dev/acl 下 Acl 项目源码，在该项目的 app/wizard 目录便是向导工具的程序源码；
- 在 acl 项目目录下编译 Acl 项目，直接运行 `make` 命令即可；
- 进入 app/wizard 目录，运行 `make`，应该会生成 wizard 可执行程序。

### 2.2、使用 wizard 向导
运行 `./wizard`，则提示如下：
```shell
$./wizard
select one below:
m: master_service; d: db; h: http; q: exit
>
```
上面提示如下：
  - `m` 生成标准（仅支持从原生 socket 读写数据）的 Acl 服务器程序；
  - `h` 生成 HTTP 服务器程序；
  - ‘d' 生成数据库相关程序（未实现）；
  - 'q' 退出 wizard 向导程序。
- 输入 `h' 表示将要生成 HTTP 服务器程序，提示用户输入将要生成的服务程序名称：
```shell
>h
please input your program name: httpd
```
此处，输入程序名 `httpd`，然后提示如下：
```shell
s: http servlet
>
```
当前，向导程序仅支持生成类似于 Java HttpServlet 类型的服务器程序，此处，我们应输入 `s`，然后提示：
```shell
create httpd/Makefile ok.
create httpd/valgrind.sh ok.
create httpd/httpd.sln ok.
create httpd/httpd.vcproj ok.
create httpd/httpd_vc2008.sln ok.
create httpd/httpd_vc2008.vcproj ok.
create httpd/httpd_vc2010.sln ok.
create httpd/httpd_vc2010.vcxproj ok.
create httpd/httpd_vc2010.vcxproj.filters ok.
create httpd/httpd_vc2012.sln ok.
create httpd/httpd_vc2012.vcxproj ok.
create httpd/httpd_vc2012.vcxproj.filters ok.
create httpd/httpd_vc2019.sln ok.
create httpd/httpd_vc2019.vcxproj ok.
create httpd/httpd_vc2019.vcxproj.filters ok.
create httpd/Makefile.in ok
create httpd/stdafx.h ok
create httpd/stdafx.cpp ok
create common_files ok!
Do you want add cookie? [y/n]:
```
上面显示 `wizard` 程序自动生成一些源文件及工程文件，接着提示是否需要支持 `http cookies`，此处我们选择 `n` -- 即不需要。然后提示：
```shell
Do you want add cookie? [y/n]: n
create httpd/http_servlet.cpp ok.
choose master_service type:
        t: for master_threads
        f: for master_fiber
        p: for master_proc
>
```
此处需要我们选择服务模型类型，此处选择 `f` 表示生成协程服务程序，显示如下：
```shell
>f
create httpd/httpd.cf ok.
create httpd/Makefile.in ok
create httpd/stdafx.h ok
create httpd/main.cpp ok
create httpd/master_service.h ok
create httpd/master_service.cpp ok
create httpd/http_service.h ok
create httpd/http_service.cpp ok
create httpd/http_servlet.h ok
create httpd/websocket.cpp ok
create httpd/websocket.h ok
create master_fiber ok!
create httpd/Makefile ok.
create httpd/Makefile.in ok
select one below:
m: master_service; d: db; h: http; q: exit
>
```
wizard显示生成了一些与协程及http服务相关的源码文件、工程文件及配置文件。至此，我们使用 wizard 生成了一个基于 Acl 协程模型的 HTTP 服务器程序，然后退出 wizard 程序。

### 2.3、快速浏览向导生成的文件
进入 `./httpd` 目录，可以看到帮我们生成了以下文件：
```shell
-rw-------  1 zsx  staff    151 May 27 10:43 Makefile
-rw-------  1 zsx  staff   4388 May 27 10:43 Makefile.in
-rw-------  1 zsx  staff   3612 May 27 10:43 http_service.cpp
-rw-------  1 zsx  staff   1327 May 27 10:43 http_service.h
-rw-------  1 zsx  staff   3032 May 27 10:43 http_servlet.cpp
-rw-------  1 zsx  staff   1336 May 27 10:43 http_servlet.h
-rw-------  1 zsx  staff   6190 May 27 10:43 httpd.cf
-rw-------  1 zsx  staff   1265 May 27 10:43 httpd.sln
-rw-------  1 zsx  staff   8492 May 27 10:43 httpd.vcproj
-rw-------  1 zsx  staff   1291 May 27 10:43 httpd_vc2008.sln
-rw-------  1 zsx  staff  10196 May 27 10:43 httpd_vc2008.vcproj
-rw-------  1 zsx  staff   1292 May 27 10:43 httpd_vc2010.sln
-rw-------  1 zsx  staff  12782 May 27 10:43 httpd_vc2010.vcxproj
-rw-------  1 zsx  staff   1944 May 27 10:43 httpd_vc2010.vcxproj.filters
-rw-------  1 zsx  staff   2036 May 27 10:43 httpd_vc2012.sln
-rw-------  1 zsx  staff  21055 May 27 10:43 httpd_vc2012.vcxproj
-rw-------  1 zsx  staff   1944 May 27 10:43 httpd_vc2012.vcxproj.filters
-rw-------  1 zsx  staff   1999 May 27 10:43 httpd_vc2019.sln
-rw-------  1 zsx  staff  22236 May 27 10:43 httpd_vc2019.vcxproj
-rw-------  1 zsx  staff   1887 May 27 10:43 httpd_vc2019.vcxproj.filters
-rw-------  1 zsx  staff   4241 May 27 10:43 main.cpp
-rw-------  1 zsx  staff   5265 May 27 10:43 master_service.cpp
-rw-------  1 zsx  staff    672 May 27 10:43 master_service.h
-rw-------  1 zsx  staff    218 May 27 10:43 stdafx.cpp
-rw-------  1 zsx  staff   3107 May 27 10:43 stdafx.h
-rwxr-xr-x  1 zsx  staff   3960 May 27 10:43 setup.sh
-rwxr-xr-x  1 zsx  staff     79 May 27 10:43 valgrind.sh
-rw-------  1 zsx  staff   2978 May 27 10:43 websocket.cpp
-rw-------  1 zsx  staff     67 May 27 10:43 websocket.h
```
可以看到，`wizard` 程序生成的文件主要包括：程序源文件、工程文件及配置文件。分别简单介绍一下这几类文件：
- 程序源文件：
  - main.cpp
  - stdafx.cpp/stdafx.h，在头文件中包含了 Acl 相关库的头文件及其它常用系统 API 头文件；
  - master_service.cpp/master_service.h，在头文件中可以看到当前的服务模型为协程服务模型，因为服务类继承于 acl::master_fiber 类；
  - http_service.cpp/http_service.h，http_servlet.cpp/http_servlet.h，与 HTTP 服务路由注册管理相关的类，一般不需要用户修改；
  - websocket.cpp/websocket.h，如果用户想要支持 websocket，可以给此类增加应用功能。
- 工程文件：
  - Makefile/Makefile.in，在 Linux/Unix 平台下的工程文件，Makefile 包含并依赖于 Makefile.in，在 Makefile.in 中可以添加编译选项、程序源文件等，在 Makefile 中可以修改程序名，如果要支持 C++11 特性，还可以打开 C++11 的编译开关；
  - xxx.sln, xxx.vcxproj, xxx.vcxproj.filters，这些文件为在 Windows 平台下使用 VS 编译示例程序的工作文件；
- 配置文件 httpd.cf，为未例服务程序运行时可以加载的配置文件。
- 安装脚本：setup.sh，可以将生成的服务器程序及配置文件安装至指定目录，如果当前系统部署了acl-master服务管理框架，则安装脚本会自动将该服务注册至acl-master的服务管理配置中。

### 2.4、编译测试协程 HTTP 服务器
在上面用向导程序生成的 `httpd` 目录下运行 `make` 编译服务程序，于是便生成了一个名为 `httpd` 的基于协程服务模型的 http 服务程序。以手工方式启动该 httpd 程序：
```shell
./httpd alone httpd.cf
```
显示如下：
```shell
listen: 127.0.0.1|8088
master_log_open(36): no MASTER_LOG's env value
src/fiber_server.cpp(1148), acl_fiber_server_main: configure file=httpd.cf
src/fiber_server.cpp(912)->server_init: can't get SERVICE_LOG's env value, use acl_master.log log
acl_inet_listen: listen 127.0.0.1:8088 ok
server_open: listen 127.0.0.1:8088 ok
master_service.cpp(126), proc_pre_jail: >>>proc_pre_jail<<<
master_service.cpp(136), proc_on_init: >>>proc_on_init<<<
master_service.cpp(142), proc_on_init: not use SSL mode
schedule event type - kernel
master_fiber.cpp(110), service_on_listen: listen 127.0.0.1|8088 ok, fd=8
master_service.cpp(131), proc_on_listen: >>>listen 127.0.0.1|8088 ok<<<
daemon started, log=acl_master.log, ev=0
```

然后我们可以用浏览器或`curl` 工具进行测试：
```shell
$ curl http://127.0.0.1:8088/test
/test/: hello world!, method=1, GET

$ curl http://127.0.0.1:8088/
hello world!
```

### 2.5、发布服务程序至acl_master服务管理框架
向导程序生成安装脚本 `setup.sh` 可用来安装编译好的服务程序，执行如下命令：
```shell
#./setup.sh /opt/soft/httpd
```
该脚本自动将 httpd 服务程序安装至 `/opt/soft/httpd/sbin/` 目录下，同时将其配置文件拷贝至 `/opt/soft/httpd/conf` 目录；如果当前环境中安装了 `acl_master` 服务管理框架，安装脚本会将该服务的配置文件全路径 `/opt/soft/httpd/conf/httpd.cf` 添加进 `acl_master` 的服务管理文件 `/opt/soft/acl-master/conf/services.cf` 中，这样当 `acl_master` 程序重启后会自动拉起在 `services.cf` 配置的程序。
