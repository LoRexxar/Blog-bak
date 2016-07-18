title: 《代码审计》一点儿笔记
date: 2015-12-21 12:41:37
tags:
- Blogs
- book_notes
categories:
- Blogs
---
![](/img/dmsj.jpg)
这两天看完《代码审计》，发现其实还是蛮有意思的，书中讲的很简单，非常易懂，看完还是记录了一些新姿势...

<!--more-->
# 配置文件

config.php cache.config.php

# 文件读取函数

file_get_contents（）、highlight_file（）、fopen（）、readfile（）、fread（）、fgetss（）、fgets（）、parse_ini_file（）、show_source（）、file（）
# 文件上传函数
move_uploaded_file（）

# xss

print、print_r、echo、printf、sprintf、die、var_dump、var_export

# 注入

## 报错注入

floor（）、updatexml（）以及extractvalue（）

GeometryCollection（）、polygon（）、GTID_SUBSET（）、multipoint（）、multilinestring（）、multipolygon（）、LINESTRING（）、exp（）

### GeometryCollection()


id = 1 AND GeometryCollection（（select * from（select * from（select user（））a）b））

### polygon()

id = 1 AND polygon（（select * from（select * from（select user（））a）b））

### multipoint()

id = 1 AND multipoint（（select * from（select * from（select user（））a）b））

### multilinestring()

id = 1 AND multilinestring（（select * from（select * from（select user（））a）b））

### multipolygon()

id = 1 AND multipolygon（（select * from（select * from（select user（））a）b））

### linestring()

id = 1 AND LINESTRING（（select * from（select * from（select user（））a）b））

### exp()

id = 1 and EXP（~（SELECT*from（SELECT user（））a））

## 二次urldecode注入

addslashes（）、mysql_real_escape_string（）、mysql_escape_string（）转义符号

urldecode或者rawurldecode函数解码

## 宽字节注入

set character_set_client=gbk

## 过滤函数

1、magic_quotes_gpc负责对GET、POST、COOKIE的值进行过滤。

2、magic_quotes_runtime对从数据库或者文件中获取的数据进行过滤。

3、addslashes函数，参数必须是string型

4、mysql_[real_]escape_string函数，转义字符串

5、intval取整函数，针对int型

# 文件包含

include（）、include_once（）、require（）和require_once（）

# 代码执行漏洞

eval（）、assert（）、preg_replace（）、call_user_func（）、call_user_func_array（）、array_map（）


call_user_func（）、call_user_func_array（）、array_map（）
usort（）、uasort（）、uksort（）、array_filter（）、array_reduce（）、array_diff_uassoc（）、array_diff_ukey（）
array_udiff（）、array_udiff_assoc（）、array_udiff_uassoc（）
array_intersect_assoc（）、array_intersect_uassoc（）
array_uintersect（）、array_uintersect_assoc（）
array_uintersect_uassoc（）、array_walk（）、array_walk_recursive（）
xml_set_character_data_handler（）、xml_set_default_handler（）
xml_set_element_handler（）、xml_set_end_namespace_decl_handler（）
xml_set_external_entity_ref_handler（）、xml_set_notation_decl_handler（）
xml_set_processing_instruction_handler（）
xml_set_start_namespace_decl_handler（）
xml_set_unparsed_entity_decl_handler（）、stream_filter_register（）
set_error_handler（）、register_shutdown_function（）、register_tick_function（）

# 命令执行漏洞

system（）、exec（）、shell_exec（）、passthru（）、pcntl_exec（）、popen（）、proc_open（）

## 命令执行过滤

1、escapeshellcmd(), 过滤整条命令.

2、escapeshellarg(),保证传入命令执行函数的参数是一字符串的形式存在的.

过滤了
'&'、'；'、'`'、'|'、'*'、'？'、'~'、'<'、'>'、'^'、'（'、'）'、'['、']'、'{'、'}'、'$'、'\'、'\x0A'、'\xFF'、’%’，'和"

# 变量覆盖漏洞

extract（）函数和parse_str（）

import_request_variables（）函数则是用在没有开启全局变量注册的时候，调用了这个函数则相当于开启了全局变量注册，在PHP 5.4之后这个函数已经被取消。

# 逻辑漏洞

1、in_array()判断是否属于数组中的一个.

2、is_numeric()判断是否一个变量是数字,这里可以通过提交hex编码直接绕过,返回ture

3、==和===,===加入了类型判断

# 特殊点

1、$_SERVER变量不受gpc保护.

2、mb_convert_encoding也有可能出现编码转换问题

3、显示错误信息需要打开php.ini中的display_errors=on或者在代码中加入error_reporting()函数,其中最常用的是E_ALL、E_WARNING、E_NOTICE、E_ALL代表提示所有问题，E_WARNING代表显示错误信息，E_NOTICE则是显示基础提示信息。

4、iconv函数编码截断,如果出现chr(128)到chr(255)之间的字符,就可以截断

5、php文件输入流

```
php：//stdin
php：//stdout
php：//stderr 
php：//input
php：//output 
php：//fd 
php：//memory
php：//temp 
php：//filter
php://input是读取post提交上来的数据,output是将数据流输出,php://filter是一个文件操作协议,效果类似readfile(),file(),file_get_contents(),
```
6、php代码解析标签

- <script language="php">...</script>

- <?..?>(需要short_open_tag=on,默认开启)

7、正则表达式没有验证开头和结尾^$

8、windows findfirstfile利用12<<可以读取123456.txt

9、php可变变量,$$可以覆盖变量,双引号中的变量可能会被解析,
```
<？php $a="${@phpinfo（）}"；？>

$a = "${ phpinfo（）}"；    @换为空格
$a = "${      phpinfo（）}"； tab
$a = "${/**/phpinfo（）}"； 
甚至回车+-!~\都可以
```
