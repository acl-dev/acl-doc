---
title: 使用 acl_master 管理你的服务器程序
date: 2023-06-10 11:00:00
tags: 服务编程
categories: 服务编程
---

# 使用 acl_master 管理你的服务器程序
## 一、概述
在 Acl(https://github.com/acl-dev/acl/) 项目中的 acl_master 模块(位于app/master/daemon/ 目录下) 是一个功能强大的服务管理模块，即可以管理使用 Acl 库编写的服务程序，也可以管理非Acl库写的服务程序（如可以管理redis, dnsmasq, ircd 等）。该模块最初来自于开源的邮件MTA--Postfix，后来被 Acl 库作者独立出来，经过长期迭代优化，最后形成一个通用的服务管理程序---acl_master，它的作用类似于 Tomcat 管理 Java Servlet 或 Apache HTTP Server 管理 CGI，管理程序与接受管理的服务子程序是紧耦合的，管理程序可以做到服务子程序的在线热升级；另外，还有一种松耦合的管理方式，比如使用 Supervisor 或 Systemd（Linux平台下）管理服务子进程，很多 Gopher 喜欢用 Supervisor 管理他们用 Go 编写的服务程序，而一些运维人员喜欢用 Systemd 管理一些服务程序。acl_master 可以支持这两类的服务管理方式。

本文主要讲解使用 acl_master 管理服务程序的一些用法。

## 二、使用 acl_master
### 2.1、项目下载
acl_master 存在于 Acl 项目中，可以从以下位置下载 Acl 项目源码：
- Github: https://github.com/acl-dev/acl
- Gitee:  https://gitee.com/acl-dev/acl

### 2.2、模块组成
- acl_master 主程序：位于 acl/app/master/daemon/ 目录下；
- 管理工具，在 acl/app/master/tools/ 下有一些辅助 acl_master 服务的管理工具：
  - master_ctl: 该工具为 acl_master 的命令行管理工具，通过与 acl_master 进程交互，实现对 acl_master 所管理的服务子进程的控制与管理；

**有了 master_ctl 命令行管理工具，技术人员就可以在 acl_master 服务管理框架下管理自己编写的应用服务了，如果感兴趣，还可以继续了解下面的几个辅助工具：**

- master_ctld: 这是一个服务程序（同样由acl_master负责管理），其与 acl_master 进行协议层的交互，并对外提供 HTTP 接口服务，实现了类似于命令行工具--master_ctl 类似的功能；
- mater_monitor：一个服务报警程序（由 acl_master 负责管理），当 acl_master 发现其所管理的服务子进程异常退出时，会通知 master_monitor 服务模块，由 master_monitor 进行异常报警（如发送报警邮件）；
- master_guard：一个定期上报 acl_master 管理服务子进程状态的服务模块，该模块定期从 acl_master 获得服务子进程的状态信息（如：内存，句柄数等）并可以将信息上报至指定数据收集模块；
- master_dispatch：该目录下的服务模块可以用来收集多个 acl_master 实例管理服务子进程的连接数信息，并可展示于 Web 浏览器上，方便管理员查看各个服务模块的连接数信息，在 master_dispatch 目录下，存在以下几个模块：
  - server：编译该目录下的程序后的可执行程序为 master_dispatch，一般与应用服务程序部署在同一个 acl_master 节点下，负责一对一地收集服务子程序的连接数信息（即一个 master_dispatch 进程负责收集一个应用服务子进程的连接数信息，将来应该需要进一步优化，使其可以收集多个服务子进程连接数信息）；
  - manager：编译后生成可执行程序 dispatch_manager，负责收集所有 master_dispatch 的上报信息，并统一缓存管理；
  - www：这是一个 Web 服务程序，可执行程序为 webapp，其会从 dispatch_manager 获取服务集群中各个服务模块的并发连接数信息，同时该模块提供的 Web 静态页面包含了可以在浏览器绘图的 Js 等静态文件，浏览器通过这些 js 脚本定期从 webapp 获得服务模块的连接数信息后，实时动态地展示在浏览器上。

### 2.3、编译安装
- 进入 acl 项目，先运行 `make` 命令编译 Acl 基础库；
- 进入 acl/app/master/daemon 目录，运行 `make` 命令编译 acl_master；
- 然后运行 `make install`，acl_master 服务程序及其配置文件便会被拷贝至 acl/dist/master/libexec/ 目录下；
- 进入 acl/app/master/tools/master_ctl 目录，运行 `make` 命令编译 acl_master 的命令行管理工具 master_ctl，然后运行 `make install`，该程序将被拷贝至 acl/dist/master/bin/ 目录下
- 进入 acl/dist/master 目录，运行安装脚本：`./setup.sh /opt/soft/acl-master`，acl_master 及其相关文件便被安装至 /opt/soft/acl-master/ 目录下；

**针对那些使用 CentOS 环境的开发及运维人员，Acl 库还提供了 RPM 包的生成及部署方式，极大的方便运维人员：** 进入 acl/packaging/ 目录，运行 `make PKG_NAME=acl_master` 便会在 x86_64/ 目录下生成 acl-master 的 RPM 安装包，用户可以直接安装该 RPM 包：`#rpm -ivh acl-master-3.5.5-0.x86_64.rpm`，最终 acl_master 及相关模块会被安装于 /opt/soft/acl-master/ 目录下。

### 2.4、运行管理
- 在 acl_master 的安装目录（/opt/soft/acl-master/sh/）下有以下几个脚本用来启停 acl_master 服务：
  - start.sh：启动 acl_master 服务；
  - stop.sh：停止 acl_master 服务；
  - master.sh（仅限Linux）：提供了 acl_master 启动、停止、重新加载配置的控制命令。
- 在 CentOS Linux 下，当以 RPM 包方式安装了 acl_master 服务后，还可以通过以下命令启停 acl_master 服务：
  - 启动服务：`#service acl-master start`；
  - 停止服务：`#service acl-master stop`。

### 2.5、使用 master_ctl 命令行管理工具
`master_ctl` 一般会被安装在 /opt/soft/acl-master/bin/ 目录下，使用该工具可以方便地管理由 acl_master 所管理的服务子进程，下面是该工具的常见功能：
```shell
sh-3.2# ./master_ctl -h
usage: ./master_ctl -h[help]
 -s master_manage_addr[default: /opt/soft/acl-master/var/public/master.sock]
 -f service_path
 -t timeout[waiting the result from master, default: 0]
 -a cmd[list|stat|start|stop|reload|restart|signal]
 -n signum[specify the signal number if command is signal]
 -e extname[specified the extname of service's path, just for start and restart]
 -v [the current version of master_ctl]
```
针对以上参数，主要说明下面几个：
- **-s** 参数指定与 `acl_master` 通信的原始套接口全路径；
- **-f** 参数指定服务子进程配置文件的全路径；
- **-a** 参数指定了常用的控制命令：
  - **list：** 列出`acl_master`所管理的所有服务子程序；
  - **start：** 启动由 **-f** 参数所指定的服务程序；
  - **stop：** 停止由 **-f** 参数所指定的服务程序；
  - **stat：** 查看由 **-f** 参数所指定的服务程序的运行状态；
  - **reload：** 使由 **-f** 参数所指定的服务程序重新加载其配置文件；
  - **restart：** 重启由 **-f** 参数所指定的服务程序；
  - **signal：** 向由 **-f** 参数所指定的服务程序发送由参数 **-n** 指定的系统信号。

## 三、小结
本文以上内容主要描述了 Acl 中服务管理框架（acl-master）的编译、安装及管理过程，为方便大家快速掌握及理解，用户不防结合另一篇文章《使用向导快速生成服务程序》https://acl-dev.cn/2023/05/27/using_wizard/ 来实际操作一下，看看 acl_master 是如何管理应用服务程序的。
