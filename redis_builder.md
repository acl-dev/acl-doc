---
title: 一个 C++ redis 集群管理工具
date: 2015-04-20 00:01
categories: redis使用
---

集群版 redis3.0 发布以来，官方仅提供了一个使用 Ruby 写的集群管理工具，在创建 redis 集群时需要使用该工具。因为 Ruby 中的一些包依赖问题，导致一些生手在建立 redis 集群时吃尽了苦头。于是 acl 库作者基于 acl 中的 redis 模块库，用 C++ 语言写了一个 redis 集群管理工具: redis_builder，没有过多的包依赖，可以方便 redis 使用者快速地建立 redis 集群，此外，该工具还可以进行一些集群的其它管理工作。

下面是 redis_builder 的一些功能：
```
./redis_build -h
usage: redis_builder.exe -h[help]
-s redis_addr[ip:port]
-a cmd[nodes|slots|create|add_node|del_node|node_id|reshard]

-p passwd
-N new_node[ip:port]
-S [add node as slave]
-f configure_file

for samples:
./redis_builder -s 127.0.0.1:6379 -a create -f cluster.xml
./redis_builder -s 127.0.0.1:6379 -a nodes
./redis_builder -s 127.0.0.1:6379 -a slots
./redis_builder -s 127.0.0.1:6379 -a del_node -I node_id
./redis_builder -s 127.0.0.1:6379 -a node_id

./redis_builder -s 127.0.0.1:6379 -a reshard
./redis_builder -s 127.0.0.1:6379 -a add_node -N 127.0.0.1:6380 -S
```

注：如果集群中的每个 redis 节点设置了密码验证，则使用该工具需要增加参数： -p [passwod]

## 一、建立 redis 集群
### 1.1、方法一：

在启动所有的 redis 进程后，可以使用 redis_builder 将这些 redis 结点组成一个 redis 集群，redis_builder 使用 xml 格式的配置文件管理 redis 各个结点的关系，如该 cluster.xml 文件的内容例如：

```xml
<?xml version="1.0"?>
<xml>
    <node addr = "192.168.136.172:16380">
        <node addr = "192.168.136.172:16381" />
        <node addr = "192.168.136.172:16382" />
    </node>
    <node addr = "192.168.136.172:16383">
        <node addr = "192.168.136.172:16384" />
        <node addr = "192.168.136.172:16385" />
    </node>
    <node addr = "192.168.136.172:16386">
        <node addr = "192.168.136.172:16387" />
        <node addr = "192.168.136.172:16388" />
    </node>
</xml>
```

这样就可以运行：./redis_builder -a create -f cluster.xml，则redis 集群便会自动建立起来，集群中主从结点的分布情况如下：

```
master: 192.168.136.172:16380
        slave: 192.168.136.172:16381
        slave: 192.168.136.172:16382
master: 192.168.136.172:16383
        slave: 192.168.136.172:16384
        slave: 192.168.136.172:16385
master: 192.168.136.172:16386
        slave: 192.168.136.172:16387
        slave: 192.168.136.172:16388
```

这种方式的好处是由配置文件直接指定集群中各个结点的主从关系，缺点是：当 redis 结点非常多时配置管理起来也非常麻烦，因此该工具提供了另外一种 redis 集群创建模式，如方法二：

### 1.2、方法二
运行方式：./redis_builder -a create -f cluster.xml -r 2，其中的 -r 参数指定每个从结点的从结点个数，当指定了 -r 参数后，配置文件中指定的主从关系便不再生效，由工具根据以下三个原则进行主从结点的自动分配：

- 主从节点在不同的服务器上
- 主节点尽量均分在各个服务器上
- 从节点尽量均匀的分在不同的机器上

如配置文件中的内容为：

```xml
<?xml version="1.0"?>
<xml>
    <node addr = "192.168.136.171:16380" />
    <node addr = "192.168.136.171:16381" />
    <node addr = "192.168.136.171:16382" />
    <node addr = "192.168.136.172:16380" />
    <node addr = "192.168.136.172:16381" />
    <node addr = "192.168.136.172:16382" />
    <node addr = "192.168.136.173:16380" />
    <node addr = "192.168.136.173:16381" />
    <node addr = "192.168.136.173:16382" />
</xml>
```

则主从结点的分布情况可能如下：

```
master: 192.168.136.171:16380
        slave: 192.168.136.172:16380
        slave: 192.168.136.173:16380
master: 192.168.136.172:16381
        slave: 192.168.136.173:16382
        slave: 192.168.136.171:16382
master: 192.168.136.173:16381
        slave: 192.168.136.172:16382
        slave: 192.168.136.171:16381
```

可以看出，主从结点的分配基本满足以上三个原则，这样的好处就是当一台服务出现问题，整个集群中的其它机器的从结点可以顺利接管主结点服务。

## 二、显示当前 redis 集群中的结点信息：
运行：`./redis_builder -s 192.168.136.171:16380 -a nodes` 显示如下信息：

```
master, id: 4dcf8df124888e614cf08bd4df7987986124dd23, addr: 192.168.136.171：16380
slots range: 5462-10922
slave, id: 2b02cda1384336956d22c6c84fbe339210959bcb, addr: 192.168.136.172:16380, master_id: 4dcf8df124888e614cf08bd4df7987986124dd23
slave, id: a041490f734ca7478330acf5b609c542936214c2, addr: 192.168.136.173:16380, master_id: 4dcf8df124888e614cf08bd4df7987986124dd23
---------------------------------------
master, id: bfa250d0f6cae39623515dbce084b904f070fd96, addr: 192.168.136.172:16381
slots range: 10923-16383
slave, id: 4dfd12785d85b8c195c899511ac470aa9a2a4181, addr: 192.168.136.173:16382, master_id: bfa250d0f6cae39623515dbce084b904f070fd96
slave, id: fff7ecf70418300e1d41c6bb572c5ba995cc44c3, addr: 192.168.136.171:16382, master_id: bfa250d0f6cae39623515dbce084b904f070fd96
---------------------------------------
master, id: ebecfb9cc3548b17b37f2de94346473baa59721c, addr: 192.168.136.173:16381
slots range: 0-5461
slave, id: 842db09336dee58bbcd13e4a93ee680d96fa9915, addr: 192.168.136.172:16382, master_id: ebecfb9cc3548b17b37f2de94346473baa59721c
slave, id: 025f58e174f8cd156d1c9621d32956bac2a90032, addr: 192.168.136.171:16381, master_id: ebecfb9cc3548b17b37f2de94346473baa59721c
```

## 三、添加新的 redis 结点

随着数据规模的扩大，如果当前 redis 集群需要添加新的 redis 结点，则可以动态扩充 redis 结点，下面提供了使用 redis_builder 增加新结点的步骤：

### 3.1、确定新结点的主结点并给新结点的主结点添加从结点

```
./redis_builder -s 192.168.136.171:16383 -N 192.168.136.172:16383 -S -a add_node
./redis_builder -s 192.168.136.171:16383 -N 192.168.136.173:16383 -S -a add_node
```

该命令确定新增结点的主结点为 192.168.136.171:16383，两个从结点为：192.168.136.172:16383 和 192.168.136.173:16383

### 3.2、将新增结点并入已有集群中

```
./redis_builder -s 192.168.136.171:16380 -N 192.168.136.171:15383 -a add_node
```

这样，便给已有的 redis 集群添加了新的结点，注意步骤 3.2 与 3.1 中参数的不同，在步骤 3.2 中将新结点添加进集群时，只需指定集群中的一个主结点即可，且不能添加 -S 参数。

在添加完新结点后， 还需要将集群中的其它结点的哈希槽及数据移至新结点上，以便达到数据均衡性。下面给出了数据迁移的过程：

## 四、将数据重新分区

运行：`./redis_builder -s 192.168.136.171:16383 -a reshard`

```
addr: 192.168.136.171:16380

id: 10280b2f2d4938c7f3dffd6a56369f470b34f11adfd
slots: 2500 - 8191
-----------------------------------------------
addr: 192.168.136.172:16381

id: 24664d58862274c8a600dae3be5c6b38998d6824
slots: 10692 - 16383
-----------------------------------------------
addr: 192.168.136.173:16381

id: 5a422ace11d20ed7b3735661bd82d7ff0343164df
slots: 0 - 2499
slots: 8192 - 10691

-----------------------------------------------
addr: 192.168.136.171:16383

id: 5a422ace11fsdfd23sdfsfsdfsfsfsfsfsfstdgdgdgf

How many slots do you want to move (from 1 to 16384) ? 2000       ------  此处指定需要迁移的哈希槽数量

What is the receiving node ID?  5a422ace11fsdfd23sdfsfsdfsfsfsfsfsfstdgdgdgf   ----- 此处指定目标 redis 结点

Please input all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots
  Type 'done' once you entered all the source node IDs.
Source node #1:
``` 

输入 all 后 redis_builder 工具会自动将数据从所有已存在结点中将哈希槽及存储于哈希槽中的数据迁移至目标结点中。

## 五、编译 redis_builder
因为该工具依赖于 lib_acl/lib_protocol/lib_acl_cpp 三个 acl 基础库，所以需要首先编译这三个库：

```
$cd lib_acl; make
$cd lib_protocol; make
$cd lib_acl_cpp; make
```

然后再进入 app/redis_tools/redis_builder 编译：
```
$cd app/redis_tools/redis_builder; make
```

## 六、参考

redis_builder 的工具下载：https://github.com/acl-dev/acl/tree/master/app/redis_tools/redis_builder

acl 中的 redis 模块例子：https://github.com/acl-dev/acl/tree/master/lib_acl_cpp/samples/redis

acl on github：https://github.com/acl-dev/acl
acl on gitee：https://gitee.com/acl-dev/acl