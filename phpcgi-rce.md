---
title: PHP CGI Windows平台远程代码执行漏洞（CVE-2024-4577）分析与复现
date: 2024-06-11 09:52:10
tags:
- php
- rce
---

在2024.6.6今天，@Orange在他的博客发布了他即将在2024年8月Black Hat USA公开的议题《**Confusion Attacks: Exploiting Hidden Semantic Ambiguity in Apache HTTP Server!**》

- https://www.blackhat.com/us-24/briefings/schedule/index.html#confusion-attacks-exploiting-hidden-semantic-ambiguity-in-apache-http-server-40227

伴随着议题的发布，今天在DEVCORE的官方博客发布了一个漏洞通报，也就是存在于**windows特殊场景下的PHP CGI远程代码执行漏洞**

- https://devco.re/blog/2024/06/06/security-alert-cve-2024-4577-php-cgi-argument-injection-vulnerability/

接下来看看漏洞的详情

<!--more-->

# 漏洞描述

**CVE-2024-4577**导致漏洞产生的本质其实是[Windows系统内字符编码转换的Best-Fit特性](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-ucoderef/d1980631-6401-428e-a49d-d71394be7da8)导致的，相对来说**PHP**在这个漏洞里更像是一个受害者。

由于Windows系统内字符编码转换的Best-Fit特性导致**PHP原本的安全限制被绕过**，再加上一些**特殊的PHP CGI环境**配置导致了这个问题，最终导致漏洞利用的算是一些PHP的小技巧。

## 影响范围

这个漏洞理论上影响PHP的所有版本

- PHP 8.3 < 8.3.8
- PHP 8.2 < 8.2.20
- PHP 8.1 < 8.1.29

除此之外的其他PHP版本官方已经不再维护了，包括PHP8.0、PHP7、PHP5在内，但是**理论上来说他们都受到这个影响。**

## 漏洞利用条件

### Windows系统内字符编码转换的Best-Fit特性

前面提到过，这个漏洞的利用前提是由于Windows系统内字符编码转换的Best-Fit特性，所以第一个前提条件就是

- 必须是**Window环境**

其次由于这个特性，windows必须使用以下其中之一的语言系统

- **繁体中文** (字码页 950)
- **简体中文** (字码页 936)
- **日文** (字码页 932)

而且除了这3个以外，其他的语言也不能完全排除影响，凡是存在该特性的的系统都受到影响。

**那什么是Best-Fit呢？或者说，为什么是Best-Fit？**

说白了其实就是windows对于不同编码字符集之间转化的一个特性，可能大家对于Best-Fit很陌生，如果换一个词叫做宽字节，我想大家就会很熟悉了，说白了就是**一些特殊字符在特殊字符集下转化就会转成一个正常的字符**

而这里用到的就是%ad

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202406110957711.png)

而这个正常的字符就是`-`，我们可以结合php的源码来看，为什么这个`-`很重要

https://github.com/php/php-src/commit/4dd9a36c16#diff-680b80075cd2f8c1bbeb33b6ef6c41fb1f17ab98f28e5f87d12d82264ca99729R1798

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202406110957598.png)

在php的代码当中其实原本就过滤了-这个符号，在新的commit当中还加入了**对0x80以上的所有字符的限制来修复**这个问题。在代码的注释当中有这么一段话来解释这个问题

```plain
Something is wrong with the XAMPP installation :-(
Apache CGI will pass the query string to the command line if it doesn't contain a '='.
This can create an issue where a malicious request can pass command line arguments to
the executable. Ideally we skip argument parsing when we're in cgi or fastcgi mode,
but that breaks PHP scripts on Linux with a hashbang: `#!/php-cgi -d option=value`.
Therefore, this code only prevents passing arguments if the query string starts with a '-'.
Similarly, scripts spawned in subprocesses on Windows may have the same issue.
```

如果get发送的请求字符串中不包含"="，那么**Apache就会把请求传到命令行作为cgi的参数**。但这会导致恶意请求就可以将命令行参数传递给php，如果直接处理传参，那么会影响到以独立脚本方式运行的PHP脚本。所以只有**当开头是-的时候(跳过所有空白符号)才阻止传递参数**。

而这个漏洞在2012年的时候就被曝光出来，就是CVE-2012-1823

https://www.leavesongs.com/PENETRATION/php-cgi-cve-2012-1823.html

当我们用%ad来代替-之后，参数传递没有被阻止，就构成了**参数注入**。

而PHP和Apache的环境就会更特殊一点儿，其实在2024年你很难找到类似的环境，但是很有趣的是，**XAMPP For Windows的默认环境就受到这个漏洞的影响。**

而相对更具体的受影响的场景有两个

### 以CGI模式运行的PHP环境

首先不得不说，这是一个非常非常少见的场景，在2024年你几乎没办法找到一个**直接以CGI模式运行的PHP环境**，而且也没人会去做这样的修改。

如果你想要改出类似的设定你需要加入以下的配置

```plain
AddHandler cgi-script .php
Action cgi-script "/cgi-bin/php-cgi.exe"
```

或者类似于

```plain
<FilesMatch "\.php$">
    SetHandler application/x-httpd-php-cgi
</FilesMatch>

Action application/x-httpd-php-cgi "/php-cgi/php-cgi.exe"
```

在XMAPP的默认配置当中，这部分代码也是被注释的，如果你想要测试这种利用**需要在httpd-xampp.conf中注释解开下面这段代码**

```plain
#
# PHP-CGI setup
#
<FilesMatch "\.php$">
    SetHandler application/x-httpd-php-cgi
</FilesMatch>
<IfModule actions_module>
    Action application/x-httpd-php-cgi "/php-cgi/php-cgi.exe"
</IfModule>
```

上面这些配置处理的都是一个类似的场景，就是apache会**把请求直接转发给php-cgi**。

结合上面的特性，你可以通过传%ad来传入一个`-`，这样在-之后的部分就会成为**php-cgi的参数**，构成**参数注入**。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202406110957918.png)

```plain
%add+allow_url_include%3don+%add+auto_prepend_file%3dphp://input
```

会变成

```plain
-d allow_url_include=on -d auto_prepend_file=php://input
```

构成参数注入导致**最终的任意代码执行**。

### 将PHP的执行程序暴露在外 - XAMPP默认配置

这个场景要特别一些，相比直接把PHP的二进制直接放在web目录下，可能更常见的还是x**ampp的默认配置。**

在httpd-xampp.conf中就可以找到这一串代码

```plain
ScriptAlias /php-cgi/ "D:/xampp/php/"
<Directory "D:/xampp/php">
    AllowOverride None
    Options None
    Require all denied
    <Files "php-cgi.exe">
          Require all granted
    </Files>
</Directory>
```

上面的配置是什么意思呢？

其实就是访问`/php-cgi/`路径的时候，会映射`D:/xampp/php/`下的文件，而这个目录下正好是php的整个目录

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202406110957534.png)

但这又有什么用呢，毕竟都denied了，**只有php-cgi.exe是granted的**，我们把视角还是回到php-cgi上。

如果我们直接访问调用php-cgi.exe，会怎么样呢？

答案是会有**安全警告**

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202406110957728.png)

那我们顺着这个思路去看下代码

https://github.com/php/php-src/blob/51379d66ec8732e506c43f6c7f1befc500117ae8/sapi/cgi/cgi_main.c#L1912

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202406110957709.png)

我们会发现我们遇到了一个新的概念叫做**force_redirect**，这其实算是**PHP的一个自我保护机制**。在P牛的知识星球里就有提过这个问题。

https://wx.zsxq.com/dweb2/index/topic_detail/15411452114282

说白了就是，PHP增加了一个配置叫做**cgi.force_redirect=1**，**开启了这个选项（默认开启）**之后，只有经过重定向的规则请求才能执行，不能直接调用执行。那有什么办法绕过呢？

其实如果顺着逻辑到这里，就已经很清晰了，第一个办法也就是最直接的办法就是，既然前面已经实现了-d修改配置，那么你就正好-d再把cgi.force_redirect修改成0就行了，非常直白。

![img](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202406110957913.png)

而第二个办法相对来说比较少见，我们继续回到源码当中可以发现，除了force_redirect的分支以外，其实如果能确保`REDIRECT_STATUS`或者`HTTP_REDIRECT_STATUS`存在值，也可以不进入到这个分支里。

![image-20240611095958631](https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/blog/202406111000460.png)

当然一般来说这种变量是不可控的，但是HTTP开头的变量一般来自于请求的头，那么我们就可以在请求头中加入`Redirect-status: 1`来设置这个变量，同样可以绕过
