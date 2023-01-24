---
title: 配置文件的读取
date: 2009-11-03 12:43
categories: 配置文件
---

配置文件的读取是程序中必要部分，虽然不算复杂，但如果每次都写配置文件的分析提取代码也是件烦人的事。现在流行的配置文件格式有：ini，xml ，简单name-value对等格式，ACL库中实现了最简单的 name-value对格式的配置文件，该文件格式有点类似于 xinetd.conf 的格式，文件格式如下：


test.cf:

```
service myapp {

    my_addr = 127.0.0.1

    my_port = 80

    my_list = www.test1.com, www.test2.com, www.test3.com, \

                   www.test4.com, www.test5.com, www.test6.com

    ...

}
```
 

其中的 "\"  是连接符，可以把折行的数据连接起来。

下面的例子读取该配置文件并进行解析：

```c
static int var_cfg_my_port;

static ACL_CFG_INT_TABLE __conf_int_tab[] = {
  /* 配置项名称, 配置项缺省值, 存储配置项值的地址, 保留字, 保留字 */
  { "my_port", 8080, &var_cfg_my_port, 0, 0 },
  { 0, 0 , 0, 0, 0 }
};

static char *var_cfg_my_addr;
static char *var_cfg_my_list;

static ACL_CFG_STR_TABLE __conf_str_tab[] = {
  /* 配置项名称, 配置项缺省值, 存储配置项值的地址 */
  { "my_addr", "192.168.0.1", &var_cfg_my_addr },
  { "my_list", "www.test.com", &var_cfg_my_list },
  { 0, 0, 0 }
};

static int var_cfg_my_check;

static ACL_CFG_BOOL_TABLE __conf_bool_tab[] = {
  /* 配置项名称, 配置项缺省值, 存储配置项值的地址 */
  { "my_check", 0, &var_cfg_my_check },
  { 0, 0, 0 }
};

void test(void)
{
  ACL_XINETD_CFG_PARSER *cfg;  // 配置解析对象

  cfg = acl_xinetd_cfg_load("test.cf");  // 读取并解析配置文件
  acl_xinetd_params_int_table(cfg, __conf_int_tab);  // 读取所有 int 类型的配置项
  acl_xinetd_params_str_table(cfg, __conf_str_tab);  // 读取所有字符串类型的配置项
  acl_xinetd_params_bool_table(cfg, __conf_bool_tab);  // 读取所有 bool 型的配置项

  acl_xinetd_cfg_free(cfg);  // 释放内存
}
```
通过调用 acl_xinetd_params_xxx_table() 函数，直接将配置项的值赋给变量，这样省去了很多麻烦。