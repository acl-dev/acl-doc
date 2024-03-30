# OpenSSL IO读写超时踩坑小记

## 一、背景

最近一个线上服务项目需要开启 SSL 功能，该项目使用了Acl库中的 SSL、HTTP及协程模块、，Acl SSL 模块封装了OpenSSL库，使用户编写 SSL 服务更加简便快捷。当该服务程序开启 SSL 功能后，发现有 TCP 连接句柄较长时间堵在 IO 读上，所设置的 SSL IO 读写超时失去作用，造成的后果是服务程序内存不断增长。因为在线连接比较大，同时再加上该服务程序采用了协程模式，使问题排查过程比较困难。

## 二、问题分析

- 项目负责人通过记录引用计数方式给出了问题可能出在 HTTPS 的读 Body 数据体部分。于是先查看 ACL 库 HTTP 模块的超时是否有效，结论正常；同时对比 HTTPS 加密 及 HTTP 明文两种方式的运行状态，结果发现 HTTPS 有连接挂起问题，而 HTTP 方式则正常，所以初步分析可能与 SSL 模块有关。

- 分析 Acl 库中 openssl_io.cpp 源码（封装了OpenSSL IO）的 read() 读部分，里面会在调用 SSL_read() 前先调用 acl_read_wait() 检查 socket 是否可读，如果读超时则返回，如果有数据可读才会调用 SSL_read() 读取数据，感觉应该没有问题，于是想借助 ChatGPT 来协助分析问题，下面是 GPT4 给出的解决 SSL 读写超时的方法：

```
OpenSSL库的IO操作本身并没有内建的超时机制，但你可以通过其它方法实现类似的效果。具体的实现方式会因你使用的环境和代码的结构而异，但以下是一种常见的方法：

将底层的socket设置为非阻塞模式，并且使用系统的函数（如`select()`, `poll()` 或者 `epoll()`）监听这些socket上的事件，并在这些函数中设置超时。

以下是一个使用`select`的示例：
```

```c
// 假设你已经设置了SSL *ssl和连接的socket fd

// 将socket设置为非阻塞模式
fcntl(fd, F_SETFL, O_NONBLOCK);

// 待超时时间设置为5秒
struct timeval timeout; 
timeout.tv_sec = 5;
timeout.tv_usec = 0;

// 对文件描述符进行监听
fd_set read_fds;
FD_ZERO(&read_fds);
FD_SET(fd, &read_fds);

// 使用select，等待socket有数据可读或者超时
int ready_for_reading = select(fd + 1, &read_fds, NULL, NULL, &timeout);
if (ready_for_reading == 1)
{
    // socket有数据可读，执行SSL_read
    int ret = SSL_read(ssl, buffer, sizeof(buffer));

    // 处理读取结果...
}
else if (ready_for_reading == 0)
{
    // select超时
    printf("SSL_read timeout.\n");
}
else
{
    // select出错
    printf("Error: select.\n");
  
    // 执行错误处理...
}
```

该方法与 Acl 库中先调用 acl_read_wait() 再调用 SSL_read() 的方式相似，另外又对比了网上查询的一些方案，思路也基本类似，

- 显然上面的方式无法在实践中解决 OpenSSL 读导致的 socket 阻塞问题，于是想，如果每次 SSL_read 时函数内部只调用一次 read 的话，上面设置的超时读方法应该是有效的，但如果 SSL_read 内部在某种情况下有多次 read 操作则前面所设置的读等待只对第一次有效，对后面的就无效了；于是又回头仔细查询 SSL_read 的帮助文档，其中有这么一段话：
```
If necessary, a read function will negotiate a TLS/SSL session, if not already explicitly performed by SSL_connect(3) or SSL_accept(3). If the peer requests a re-negotiation, it will be performed transparently during the read function operation. The behaviour of the read functions depends on the underlying BIO.
```
通过这段话，隐约感觉 SSL_read 内部可能存在多次 read 过程，但上面的话还有点晦涩，接下来只能去看 SSL_read 源码了，从 ssl_lib.c 中找到 SSL_read，其读数据的大体流程为：
`SSL_read` >> `ssl_read_internal` >> `s->method->ssl_read`，然后再顺藤摸瓜，找到 ssl_read 的具体位置在 s3_lib.c 的 ssl3_read_internal 函数中，如下：
```c
static int ssl3_read_internal(SSL *s, void *buf, size_t len, int peek,
                              size_t *readbytes)
{
    int ret;

    clear_sys_error();
    if (s->s3->renegotiate)
        ssl3_renegotiate_check(s, 0);
    s->s3->in_read_app_data = 1;
    ret =
        s->method->ssl_read_bytes(s, SSL3_RT_APPLICATION_DATA, NULL, buf, len,
                                  peek, readbytes);
    if ((ret == -1) && (s->s3->in_read_app_data == 2)) {
        /*
         * ssl3_read_bytes decided to call s->handshake_func, which called
         * ssl3_read_bytes to read handshake data. However, ssl3_read_bytes
         * actually found application data and thinks that application data
         * makes sense here; so disable handshake processing and try to read
         * application data again.
         */
        ossl_statem_set_in_handshake(s, 1);
        ret =
            s->method->ssl_read_bytes(s, SSL3_RT_APPLICATION_DATA, NULL, buf,
                                      len, peek, readbytes);
        ossl_statem_set_in_handshake(s, 0);
    } else
        s->s3->in_read_app_data = 0;

    return ret;
}
```
从上面代码可以看出存在两处读操作（调用ssl_read_bytes），再跟踪一下 ssl_read_bytes 应该就可以找到系统读 API了，懒得跟踪了。即然坐实了可能存在两次 read 的过程，则就可以认定在 SSL_read 设置的读等待超时对于第二次读是无效的。 

## 三、解决方案

问题原因分析出来了，但必须得要有解决方案才成，毕竟生产环境中的项目需要等米下锅。