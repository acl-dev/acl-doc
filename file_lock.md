---
title: Unix 系统文件锁踩坑记
date: 2024-04-27 13:00
categories: 文件
---

# Unix 系统文件锁踩坑记
## 1、背景

最近的一个项目需要通过文件锁方式实现多进程之间的资源互斥，但却遇到一个诡异的现象：当进程数比较少时，感觉加锁是正常的，但当进程数一多，加锁就失效了，似乎多个进程都可以同时加一把锁，锁的互斥性完全失效。因为锁功能的实现经过Acl库（https://github.com/acl-dev/acl ）进行了二次封装，同时项目本身又比较复杂，所以决定做一个简单的例子测试一下原因。

## 2、问题分析

一般来讲，编写简单示例来复现线上系统问题并不是一件容易的事，毕竟线上系统模块众多而且复杂，有时很难知道哪些模块会导致问题发生。为了复现线上问题，专门写了一个简单的demo（参见：https://github.com/acl-dev/demo/blob/master/c/lock/file_lock.c ），通过对比测试不同条件加锁行为最终复现了线上系统问题，并最终得以解决。

在 Unix 平台上对文件进行加锁时一般会使用 fcntl 系统API，示例代码如下：

```c
int fcntl_lock(const char *filename) {
	int fd = open(filename, O_RDWR | O_CREAT, 0600);
	struct flock lock;

	if (fd == -1) {
		return -1;
	}

	lock.l_type   = F_WRLCK;
	lock.l_whence = SEEK_SET;
	lock.l_start  = 0;
	lock.l_len    = 0;

	int ret = fcntl(fd, F_SETLKW, &lock);
	...
}
```

单独测试这段代码是没有问题的：当第一个进程使用上述方法对指定文件加锁后，其它进程无法采用同样的方式再次加锁。

但在加锁成功后，如果再次打开该文件然后关闭，则就发生了预料之外的问题，针对上述代码稍作修改：

```c
	...
	int ret = fcntl(fd, F_SETLKW, &lock);
	if (ret == -1) {
		return -1;
	}

	int fd2 = open(filename, O_RDONLY, 0600);
	if (fd2 >= 0) {
		close(fd2);
	}
	...
```

对修改后（对同一文件进行第二次打开并`关闭`）的加锁代码进行测试时发现`加锁互斥`作用失效（即多个进程均可以对同一文件进行加锁）；另外，如果仅有二次打开并没有关闭，则文件锁的互斥行为依然有效，看来问题出在第二次打开文件并`关闭`后（有可能在关闭后内核的某些行为使 `fcntl` 加锁文件失效）。

编译上面 file_lock.c 源码生成 file_lock 可执行程序，然后以 `./file_lock -o -c` 方式启动两个进程，会发现两个进程均可以正常加锁，表明文件加锁失效。

## 3、问题解决

既然使用 `fcntl` API 加锁文件存在以上缺陷，则需要采用其它方法避免这一问题。Unix 系统提供了 `flock` API 对文件进行加锁互斥。使用 flock 加锁文件的示例如下：

```c
int flock_lock(const char *filename) {
	int fd = open(filename, O_RDWR | O_CREAT, 0600);
	if (fd == -1) {
		return -1;
	}

	int ret = flock(fd, LOCK_EX);
	if (ret == -1) {
		return -1;
	}

	int fd2 = open(filename, O_RDONLY, 0600);
	if (fd2 >= 0) {}
		close(fd2);
	}
	...
}
```

通过测试发现，在调用 `flock` 后再对同一文件进行『打开、关闭』操作，文件锁依然是有效的（即只有第一个进程可以加锁成功，后续进程无法再次加锁）。看来 `flock` 解决了 `fcntl` 对文件加锁时遇到的问题。

为了避免用户在使用文件锁时遇到类似的坑，在 Acl 中对文件锁进行了封装，并在内部优先使用 `flock` 方式加锁。可以参考示例：https://github.com/acl-dev/demo/blob/master/c%2B%2B/file/file_lock.cpp 。

## 4、小结

看似一个简单的小问题却耗费了1，2天时间进行分析，这进一步加深了自己对文件锁的理解。另外一方面，知识经验是需要长期积累的，与智商无关，因为有些知识点本就是一层窗户纸：会就是会，不会就是不会。