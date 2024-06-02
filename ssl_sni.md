# 使用 SSL SNI 预防 SSL 握手攻击

## 一、概述

SSL/TLS是用于在网络上进行安全通信的加密协议，各网络公司为保证数据传输的安全性均采用了SSL/TLS通信方式，相校于明文传输方式，SSL/TLS对于计算成本要求更高，尤其是在SSL/TLS握手阶段更是耗费了大量计算资源，攻击者可以轻易利用这一问题对服务端发起SSL/TLS握手攻击，攻击者只需使用少量的廉价攻击源（不必象流量攻击那样需要大量的攻击源）便可发起SSL/TLS握手攻击，占用服务端大量的CPU资源。

如何有效地防止此类握手攻击呢？想到了在SSL SNI交互阶段进行防御的方案。

SSL SNI（Server Name Indication，服务器名称指示）是TLS协议（传输层安全协议）的一个扩展，它允许在握手阶段向服务器指明客户端正在请求哪个主机名。这样，服务器可以根据客户端提供的主机名返回适当的SSL证书，从而使得在同一个IP地址上托管多个安全网站成为可能。

在 SSL SNI 交互阶段，数据是明文传输的，不需要浪费服务端多少计算资源，我们可以在 SSL SNI 阶段对 SSL/TLS 客户端进行身份验证并进行握手攻击拦截。

## 二、关于SSL SNI

在没有SNI之前，如果在同一个IP地址上托管多个SSL/TLS网站，服务器在SSL握手时无法知道客户端想要访问哪个网站，从而无法发送正确的SSL证书。这会导致SSL握手失败或返回错误的证书。

SNI的作用流程大致如下：
- 1. 客户端发起连接，并在SSL/TLS握手的初始阶段包含它想要连接的主机名（在ClientHello消息中）。
- 2. 服务器接收到ClientHello消息，读取其中的主机名信息。
- 3. 服务器根据这个主机名选择并发送对应的SSL证书。
- 4. 后续的握手过程按照通常的SSL/TLS流程进行。

通过SNI，多个SSL/TLS站点可以在同一个服务器或同一个IP地址上共存，而且每一个站点都可以拥有自己独立的证书。这对于节省IP地址和简化服务器配置有重要意义。

## 三、实施案例

Acl 库封装的 SSL 模块中提供了通过 SSL SNI 验证客户端身份的能力，下面给出了操作过程：

### 3.1、SSL 服务端

#### 3.1.2、SSL服务端实现SSL SNI校验类

下面给出 Acl SSL 模块中 SSL SNI 验证基类声明：

```c++
namespace acl {

class ACL_CPP_API ssl_sni_checker {
public:
	ssl_sni_checker() {}
	virtual ~ssl_sni_checker() {}

	/**
	 * 虚方法用来检查输入的sni host是否合法，子类必须实现
	 * @param sni {const char*} 客户端传来的 sni 字段
	 * @param host {acl::string&} 从 sni 中提取的 host 字段
	 * @return {bool} 检查是否合法
	 */
	virtual bool check(sslbase_io* io, const char* sni, string& host) = 0;
};

}
```
用户需要实现 SSL SNI 子类并完成其中的纯虚方法，在该方法完成对客户端身份的校验。下面给出简单示例：

```c++
class my_sni_checker : public acl::ssl_sni_checker {
public:
        my_sni_checker() {}
        ~my_sni_checker() {}

        // @override
        bool check(acl::sslbase_io* io, const char* sni, acl::string& host) {
                if (sni == NULL || *sni == 0) {
                        // 如果客户端未提供 SNI 标识字符串则返回错误，禁止进行 SSL 握手. 
                        printf("No SNI=%p\r\n", sni);
                        return false;
                }

                // 从 SNI 中提取服务端主机域名，并保存，以便 Acl SSL 模块选择本地的 SSL 证书
                // 并进行 SSL 握手。
                acl::string buf(sni);
                const std::vector<acl::string>& tokens = buf.split2("|");
                if (tokens.size() != 2) {
                    printf("Invalid sni=%s\r\n", sni);
                    return false;
                }

                host = tokens[1];

                // check_token(tokens[0]); // 应用自行检查加密串的合法性。

                printf("Check sni ok, sni=%s, host=%s\r\n", sni,, host.c_str());

                // 返回 true 则允许 Acl SSL 模块可以进一步进行 SSL 握手。
                return true;
        }
};
```

SSL 客户端模块与服务端可以协商一个 sni 的生成规则，比如 sni 字符串类似于 `token|host`（比如：xxxxxx|myhost.test.com)，期中 token 字段可以是双方私密生成的加密字符串，host 为主机域名。服务端仅针放行符合加密规则的客户端 SSL 连接，其它连接一概拦截。

#### 3.1.2、创建僵尸 SSL 对象中添加 SNI 验证对象

下面代码是创建全局的 SSL 配置管理对象代码：
```c++
acl::sslbase_conf* ssl_conf =  new acl::openssl_conf(true);
```

全局 SSL 配置管理对象创建后，添加 SSL SNI 验证对象：

```c++
ssl_conf->set_sni_checker(new ssl_sni_checker());
```

#### 3.1.3、与 SSL 客户端进行 SSL 握手

与客户端连接进行 SSL 握手：

```c++

bool ssl_handshake(acl::sslbase_conf& ssl_conf, acl::socket_stream& conn) {
    acl::sslbase_io* ssl = ssl_conf.create(false);
    if (conn.setup_hook(ssl) == ssl) {  // SSL 握手失败，包括 SSL SNI 查检失败。
        ssl->destroy();
        return false;
    }
    return true;
}
```

### 3.2、SSL 客户端

客户端在与服务端建立 TCP 连接后，进行 SSL 握手时需要先设置 SSL SNI 字段，SNI 字段中既有域名信息，又有加密串信息，当服务端收到该 SNI 串时，从中取出加密及域名信息后，先验证加密数据的合法性，验证通过则进行 SSL 握手，否则禁止 SSL 握手，从而避免浪费 CPU 计算资源。下面是 SSL 客户端的简单示例：

```c++

bool ssl_handshake(acl::sslbase_conf& conf, acl::socket_stream& conn) {
    acl::sslbase_io* ssl = conf.create(false);
    const char* sni = "crypted_token|mytest.com";
    ssl->set_sni_host(sni);  // 设置 SNI 字段
    if (conn.setup_hook(ssl) == ssl) {
        // SSL 握手失败，销毁 SSL IO 对象。
        ssl->destroy();
        return false;
    }
    return true;
}
```

### 3.3、参考

- 客户端示例：acl/lib_acl_cpp/samples/ssl/client
- 服务端示例：acl/lib_acl_cpp/samples/ssl/server
- 使用SSL中对数据进行加密传输：https://acl-dev.cn/2020/01/15/ssl/
