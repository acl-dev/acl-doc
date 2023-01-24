---
title: ACL编程之父子进程机制，父进程守护子进程以防止子进程异常退出
date: 2009-06-07 13:45
categories: 进程控制
---

在WIN32平台进行编程时，经常会遇到工作进程因为程序内部BUG而异常退出的现象，当然为了解决此类问题最好还是找到问题所在并解决它，但如果这类导致程序崩溃的BUG并不是经常出现，只有当某种条件发生时才会有，在我们解决BUG的时间里，为了尽最大可能地为用户提供服务可以采用一种父进程守护机制：当子进程异常退出时，守护父进程可以截获这一消息，并立即重启子进程，这样用户就可以继续使用我们的程序了，当然如果子进程的问题比较严重频繁地 DOWN掉，而父进程却不停地重启子进程的话，势必造成用户机系统资源的大量耗费，那我们的程序就如病毒一样，很快耗尽了用户机资源，所以需要父进程能够智能地控制重启子进程的时间间隔。
本文将给出一个具体的例子（利用ACL库），介绍父、子进程的编程方法。

## 一、接口介绍
### 1.1 以守护进程方式运行的接口
创建守护进程的方式非常简单，只需要调用 acl_proctl_deamon_init, acl_proctl_daemon_loop 两个函数即可
接口说明如下：

```c
/**
 * 初始化进程控制框架（仅 acl_proctl_start 需要）
 * @param progname {const char*} 控制进程进程名
 */
ACL_API void acl_proctl_deamon_init(const char *progname);

/**
 * 控制进程作为后台服务进程运行，监视所有子进程的运行状态，
 * 如果子进程异常退出则会重启该子进程
 */
ACL_API void acl_proctl_daemon_loop(void);
```

### 1.2 以命令方式来控制守护进程（守护进程即控制进程的意思）
守护进程启动后，可以以命令方式控制守护进程来启动、停止子进程，或查询显示当前正在运行的子进程。
启动子进程：acl_proctl_start_one
停止子进程：acl_proctl_stop_one
停止所有子进程：acl_proctl_stop_all
查询子进程是否在运行：acl_proctl_probe
查询当前所有在运行的子进程：acl_proctl_list
通过守护进程停止所有子进程且守护进程自身退出：acl_proctl_quit

接口说明如下：

```c
/**
 * 以命令方式启动某个子进程
 * @param progname {const char*} 控制进程进程名
 * @param progchild {const char*} 子进程进程名
 * @param argc {int} argv 数组的长度
 * @param argv {char* []} 传递给子进程的参数
 */
ACL_API void acl_proctl_start_one(const char *progname,
    const char *progchild, int argc, char *argv[]);

/**
 * 以命令方式停止某个子进程
 * @param progname {const char*} 控制进程进程名
 * @param progchild {const char*} 子进程进程名
 * @param argc {int} argv 数组的长度
 * @param argv {char* []} 传递给子进程的参数
 */
ACL_API void acl_proctl_stop_one(const char *progname,
    const char *progchild, int argc, char *argv[]);

/**
 * 以命令方式停止所有的子进程
 * @param progname {const char*} 控制进程进程名
 */
ACL_API void acl_proctl_stop_all(const char *progname);

/**
 * 探测某个服务进程是否在运行
 * @param progname {const char*} 控制进程进程名
 * @param progchild {const char*} 子进程进程名
 */
ACL_API void acl_proctl_probe(const char *progname, const char *progchild);

/**
 * 列出当前所有正在运行的服务进程
 * @param progname {const char*} 控制进程进程名
 */
ACL_API void acl_proctl_list(const char *progname);

/**
 * 以命令方式通知控制进程停止所有的子进程，并在子进程退出后控制进程也自动退出
 * @param progname {const char*} 控制进程进程名
 */
ACL_API void acl_proctl_quit(const char *progname);
```

### 1.3、子进程编写
子进程编程也比较容易，只需在程序初始化时调用　acl_proctl_child　即可，这样子进程就会在硬盘创建自己的信息并与父进程（即守护进程）建立联系。
接口说明：

```c
/**
 * 子进程调用接口，通过此接口与父进程之间建立控制/被控制关系
 * @param progname {const char*} 子进程进程名
 * @param onexit_fn {void (*)(void*)} 如果非空则当子进程退出时调用的回调函数
 * @param arg {void*} onexit_fn 参数之一
 */
ACL_API void acl_proctl_child(const char *progname, void (*onexit_fn)(void *), void *arg);
```

## 二、例子
### 2.1、父进程
程序名：acl_project\samples\proctl\proctld.cpp

```c
// proctld.cpp : 定义控制台应用程序的入口点。
//
#pragma comment(lib,"ws2_32")
#include "lib_acl.h"
#include <assert.h>

static void init(void)
{
	acl_init();  // 初始化ACL库  
}

static void usage(const char *progname)
{
	printf("usage: %s -h [help] -d [START|STOP|QUIT|LIST|PROBE] -f filepath -a args\r\n",
			progname);
	getchar();
}

int main(int argc, char *argv[])
{
	char  ch, filepath[256], cmd[256];
	char **child_argv = NULL;
	int   child_argc = 0, i;
	ACL_ARGV *argv_tmp;

	filepath[0] = 0;
	cmd[0] = 0;

	init();

	while ((ch = getopt(argc, argv, "d:f:a:h")) > 0) {
		switch(ch) {
			case 'd':
				ACL_SAFE_STRNCPY(cmd, optarg, sizeof(cmd));
				break;
			case 'f':
				ACL_SAFE_STRNCPY(filepath, optarg, sizeof(filepath));
				break;
			case 'a':
				argv_tmp = acl_argv_split(optarg, "|");
				assert(argv_tmp);
				child_argc = argv_tmp->argc;
				child_argv = (char**) acl_mycalloc(child_argc + 1, sizeof(char*));
				for (i = 0; i < child_argc; i++) {
					child_argv[i] = acl_mystrdup(argv_tmp->argv[i]);
				}
				child_argv[i] = NULL;

				acl_argv_free(argv_tmp);
				break;
			case 'h':
				usage(argv[0]);
				return (0);
			default:
				usage(argv[0]);
				return (0);
		}
	}

	if (strcasecmp(cmd, "STOP") == 0) {
		// 向守护进程发送消息命令，停止某个子进程或所有的子进程
		if (filepath[0])
			acl_proctl_stop_one(argv[0], filepath, child_argc, child_argv);
		else
			acl_proctl_stop_all(argv[0]);
	} else if (strcasecmp(cmd, "START") == 0) {
		if (filepath[0] == 0) {
			usage(argv[0]);
			return (0);
		}
		// 向守护进程发送消息命令，启动某个子进程
		acl_proctl_start_one(argv[0], filepath, child_argc, child_argv);
	} else if (strcasecmp(cmd, "QUIT") == 0) {
		// 向守护进程发送消息命令，停止所有的子进程同时守护父进程也退出
		acl_proctl_quit(argv[0]);
	} else if (strcasecmp(cmd, "LIST") == 0) {
		// 向守护进程发送消息命令，列出由守护进程管理的正在运行的所有子进程
		acl_proctl_list(argv[0]);
	} else if (strcasecmp(cmd, "PROBE") == 0) {
		if (filepath[0] == 0) {
			usage(argv[0]);
			return (0);
		}
		// 向守护进程发送消息命令，探测某个子进程是否在运行
		acl_proctl_probe(argv[0], filepath);
	} else {
		// 父进程以守护进程方式启动
		char  buf[MAX_PATH], logfile[MAX_PATH], *ptr;

		// 获得父进程执行程序所在的磁盘路径
		acl_proctl_daemon_path(buf, sizeof(buf));
		ptr = strrchr(argv[0], '\\');
		if (ptr == NULL)
			ptr = strrchr(argv[0], '/');

		if (ptr == NULL)
			ptr = argv[0];
		else
			ptr++;

		snprintf(logfile, sizeof(logfile), "%s/%s.log", buf, ptr);
		// 打开日志文件
		acl_msg_open(logfile, "daemon");
		// 打开调试信息
		acl_debug_init("all:2");

		// 以服务器模式启动监控进程
		acl_proctl_deamon_init(argv[0]);
		// 父进程作为守护进程启动
		acl_proctl_daemon_loop();
	}

	if (child_argv) {
		for (i = 0; child_argv[i] != NULL; i++) {
			acl_myfree(child_argv[i]);
		}
		acl_myfree(child_argv);
	}
	return (0);
}
```

### 2.2、子进程
acl_project\samples\proctl\proctlc.cpp

```c
// proctlc.cpp : 定义控制台应用程序的入口点。
//
#pragma comment(lib,"ws2_32")
#include "lib_acl.h"

static void onexit_fn(void *arg acl_unused)
{
	printf("child exit now\r\n");
}

int main(int argc, char *argv[])
{
	int   i;

	acl_socket_init();
	acl_msg_open("debug.txt", "proctlc");
	acl_msg_info(">>> in child progname(%s), argc=%d\r\n", argv[0], argc);
	if (argc > 1)
		acl_msg_info(">>> in child progname, argv[1]=(%s)\r\n", argv[1]);

	// 子进程启动，同时注册自身信息
	acl_proctl_child(argv[0], onexit_fn, NULL);

	for (i = 0; i < argc; i++) {
		acl_msg_info(">>>argv[%d]:%s\r\n", i, argv[i]);
	}

	i = 0;
	while (1) {
		acl_msg_info("i = %d\r\n", i++);
		if (i == 5)
			break;
		else
			sleep(1);
	}
	return (-1);  // 返回 -1 是为了让父进程继续启动
}
```

### 2.3、编译、运行
可以打开 acl_project\win32_build\vc\samples\samples_vc2003.sln，编译其中的 proctlc, proctld 两个工程，便会生成两个可执行文件：proctlc.exe(子进程程序），proctld.exe(父进程程序）。
先让父进程以守护进程模式启动 proctld.exe，然后运行 proctld.exe -d START {path}/proctlc.exe 通知父进程启动子进程；可以运行 proctld.exe -d LIST 列出当前正在运行的子进程，运行 proctld.exe -d PROBE {path}/proctld.exe 判断子进程是否在运行，运行 proctld.exe -d STOP {path}/proctld.exe 让守护父进程停止子进程，运行 proctld.exe -d QUID 使守护进程停止所有子进程并自动退出。
另外，从子进程的程序可以看出，每隔5秒子进程就会异常退出，则守护进程便会立即重启该子进程，如果子进程死的过于频繁，则守护进程会延迟重启子进程，以防止太过耗费系统资源。

## 三、小结
因为有守护进程保护，就不必担心子进程（即你的工作进程）异常崩溃了，这种父子进程模型可以应用于大多数工作子进程偶尔异常崩溃的情形，如果你的程序 BUG太多，每一会儿就崩溃好多次，建议你还是先把主要问题解决后再使用父子进程，毕竟如果你的程序太过脆弱，虽然父进程能不断地重启你的程序，但你还是不能为用户提供正常服务。这种模型适用于在WIN32平台下，你的程序可能写得比较复杂，程序基本上是比较健壮的，只是会因偶尔某些原因而异常退出的情况。

github：https://github.com/acl-dev/acl
gitee：https://gitee.com/acl-dev/acl