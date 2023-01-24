---
title: 使用 redis_builder 管理 redis 集群
date: 2016-04-23 19:59
categories: redis使用
---

在 <一个 C++ redis 集群管理工具> 中主要讲述了如何使用 redis_builder 工具创建 redis 集群的过程，除此之外，该工具还具有更为强大的 redis 客户端管理功能（相对于官方提供的 redis-cli功能要强大的多）。本文主要讲解使用 redis_builder 以交互方式管理 redis 集群的过程。

## 一、启动 redis_builder 的交互过程

只要运行：./redis_builder -s redis_ip:redis_port，便进入 redis_builder 工具的命令行交互过程，其中 redis_ip 和 redis_port 为 redis 集群中的任一个节点的 IP 及端口号。启动成功后，显示界面如下：

![redis_builder界面](/img/redis_builder.png)

在上图中，首先显示了针对一些 redis 客户端命令的：是否采用广播方式，命令执行权限。

“广播方式”的含义是：该命令需要向 redis 集群中的所有节点发送某个 redis 客户端命令，有的命令向所有主从节点广播，有的命令仅向所有主节点广播。对于象 config xxx xxx （如：config get save）类的命令，是需要向所有主从节点广播以获得所有节点的配置信息；对于象 dbsize 命令，则需要向所有主节点广播，以便获得整个集群总的键值数。

“执行权限”的用处是：1、防止误操作，2、禁止使用 redis_builder 执行某些非常危险的命令。其中“执行权限”的三个级别分别为：yes --- 允许执行，warn --- 警告执行，no --- 禁止执行。对于大部分命令，如：get, incr, dbsize 等命令的执行对于整个系统一般是无害的，所以不会有限制，而对于象 del 类的命令则带有一定危险性，管理员可以配置该命令（下面单独介绍如何配置命令的执行权限）的执行为“警告”方式，这样当运行此类命令时，redis_builder 会首先提示警告信息让管理员再次确认是否真的要执行，这样做的好处是可以在一定程度上防止管理员误操作。

## 二、配置 redis 客户端命令的执行权限
当以命令行交互方式启动 redis_builder 时，该工具首先会强制检查该工具所在目录下是否有 redis_commands.txt 文件（该文件里配置着一些 redis 客户端命令的执行权限及广播方式），如果存在，则加载命令配置信息，如果该文件不存在则尝试加载由 -F 指定的命令配置信息。之所以要强制加载 redis_commands.txt 命令配置文件，主要适应这样一种应用场景：由 root 超级管理员创建了 redis_commands.txt 文件（属主当然为 root），普通用户只有读的权限，这样 root 管理员在 redis_commands.txt 中设置的 redis 命令执行权限普通用户是不能修改的。

如果在 redis_builder 命令的相同目录下存在 redis_commands.txt 文件且内容如下：
![redis命令权限](/img/redis_commands.png) 

则启动 redis_builder 交互方式后，则显示一些 redis 命令的执行权限如下（红圈内的命令为 redis_commands.txt 文件指定的）：

i![redis命令权限](/img/redis_commands2.png)

如果在 redis_commands.txt 指定的 redis 命令与 redis_builder 内部缺省的命令相同，则执行 redis_commands.txt 中指定的 redis 命令权限。

## 三、redis_builder 在交互方式下的命令操作
在交互模式下，redis_builder 可以执行所有的 redis 客户端命令，其中，redis_builder 内置了一些针对某些重要命令的特殊执行过程，除了这些内置的 redis 客户端命令，其它的命令执行方式均为标准的 redis 协议命令格式（具体格式可参考：redis.io 或 redisdoc.com）。下面首先介绍 redis_builder 内置的一些 redis 命令，随便输入一些字符串回车后，显示内置的命令如下：

![redis命令帮助](/img/redis_help.png)

- keys 命令：该命令根据字符串匹配模式 pattern 扫描 redis 集群中所有子节点（之所以扫描子节点，是为了减轻对主节点的访问压力，如果需要强制扫描主节点，则在启动 redis_builder 时增加参数 -M），获得符合匹配条件的所有键值，为了防止键太多造成刷屏问题，可以通过 limit 参数指定每个节点显示的条目数
- get 命令：该命令可以获得五种数据类型（STRING, HASH, LIST, SET, ZSET）的值，内部会根据数据类型自动匹配查询，并根据类型的不同按不同的方式显示。示例如下：

![redis获取命令](/img/redis_get.png)

由上图可以看到，对于 STRING 和 HASH 类型，get 命令会自行进行分别处理。此外，为了防止结果集太大造成刷屏，还可以通过指定参数 :limit 来限定显示的条目，如对于 HASH 类型的 hash_test_key_53479，可以这样查找：get :2 hash_test_key_53479，虽然有三个值，则最多只会显示两个，如下：
![redisHash获取命令](/img/redis_hash_get.png)

- dbsize 命令：这是一个广播式式命令，会向所有主节点（当启动时指定参数 -M）或从所有主节点各挑选一个子节点发送 dbsize，获得整个 redis 集群中总的记录条数：

![redis数据量命令](/img/redis_dbsize.png)

- nodes 命令：该命令会发送给随意一个 redis 节点，获得当前 redis 集群的分布状态，显示如下：
![redis节点状态命令](/img/redis_nodes.png)

该图以树型方式展示了整个 redis 集群中的主从分布情况，层次结构清晰，一目了然。

- server 命令：该命令用于切换 redis 集群，管理员可以使用该命令随时切换至其它的 redis 的集群，从而方便管理员使用 redis_builder 命令同时管理多个 redis 集群。命令格式：server ip port
- 非 redis_builder 内置的其它 redis 命令：随了上面列出的一些内置命令外，管理员可以通过在命令行中输入任意合法的 redis 客户端命令，redis_builder 会将该命令发送至单个 redis 节点或广播（通过在 redis_commands.txt 指定广播方式）至所有的节点。比如：

![redis设置命令](/img/redis_set.png)

图中的 set 及 hmset 命令均不是 redis_builder 内置的命令，但 redis_builder 依然可以按标准 redis 协议命令将请求发送至 redis 节点。

## 四、使用 redis_builder 实时显示当前 redis 集群的运行状态：
此外，redis_builder 还提供了以命令行方式实时显示当前 redis 集群的运行状态，显示如下：
![redis状态](/img/redis_status.png)
 
## 五、编译 redis_builder
因为该工具依赖于 lib_acl/lib_protocol/lib_acl_cpp 三个 acl 基础库，所以需要首先编译这三个库：

```
$cd lib_acl; make
$cd lib_protocol; make
$cd lib_acl_cpp; make
```

然后再进入 app/redis_tools/redis_builder 编译：$cd app/redis_tools/redis_builder; make

六、参考

redis_builder 工具下载：https://github.com/acl-dev/acl/tree/master/app/redis_tools/redis_builder
acl 中的 redis 模块例子：https://github.com/acl-dev/acl/tree/master/lib_acl_cpp/samples/redis

acl on github：https://github.com/acl-dev/acl
acl on gitee：https://gitee.com/acl-dev/acl