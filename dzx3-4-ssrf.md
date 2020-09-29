---
title: Discuz x3.4前台SSRF分析
date: 2018-12-07 18:12:56
tags:
- ssrf
---
2018年12月3日，@L3mOn公开了一个Discuz x3.4版本的前台SSRF，通过利用一个二次跳转加上两个解析问题，可以巧妙地完成SSRF攻击链。

[https://www.cnblogs.com/iamstudy/articles/discuz_x34_ssrf_1.html](https://www.cnblogs.com/iamstudy/articles/discuz_x34_ssrf_1.html)

在和小伙伴@Dawu复现过程中发现漏洞本身依赖多个特性，导致可利用环境各种缩减和部分条件的强依赖关系大大减小了该漏洞的危害。后面我们就来详细分析下这个漏洞。

<!--more-->

# 漏洞产生条件

- 版本小于 41eb5bb0a3a716f84b0ce4e4feb41e6f25a980a3
[补丁链接](https://gitee.com/ComsenzDiscuz/DiscuzX/commit/41eb5bb0a3a716f84b0ce4e4feb41e6f25a980a3)
- PHP版本大于PHP 5.3
- php-curl <= 7.54
- DZ运行在80端口
- 默认不影响linux（未100%证实，测试常见linux环境为不影响）

# 漏洞复现

## ssrf

首先漏洞点出现的位置在`/source/module/misc/misc_imgcropper.php line 55`

![image.png-31.8kB][1]

这里`$prefix`变量为`/`然后后面可控，然后进入函数里

`/source/class/class_image.php line 52 Thumb函数`

![image.png-83.2kB][2]

然后跟入init函数（line 118）中
![image.png-53kB][3]

很明显只要`parse_url`解得出host就可以通过dfsockopen发起请求

由于这里前面会补一个`/`，所以这里的source必须是`/`开头，一个比较常见的技巧。

```
//baidu.com
```

这样的链接会自动补上协议，至于怎么补就要看具体的客户端怎么写的了。

我们接着跟`dfsockopen`到`/source/function/function_core.php line 199`
![image.png-33.4kB][4]

然后到`source/function/function_filesock.php line 31`
![image.png-112.5kB][5]

主要为红框部分的代码，可以看到请求的地址为`parse_url`下相应的目标。

由于前面提到，链接的最前为`/`，所以这里的`parse_url`就受到了限制。

![image.png-121kB][6]

由于没有`scheme`，所以最终curl访问的链接为
```
://google.com/
```
前面自动补协议就成了
```
http://://google.com/
```
这里就涉及到了很严重的问题，就是对于curl来说，请求一个空host究竟会请求到哪里呢？

在windows环境下,libcurl版本`7.53.0`
```
![image.png-99.6kB][7]
```

可以看到这里请求了我本机的ipv6的ip。

在linux环境（ubuntu）下，截图为`7.47.0`

![image.png-46.6kB][8]

测试了常见的各种系统，测试过程中没有找到不会报错的curl版本，暂时认为只影响windows的服务端环境。

再回到代码条件下，可以把前面的条件回顾一下：
1、首先我们需要保证`/{}`可控在解`parse_url`操作下存在host。

要满足这个条件，我们首先要对`parse_url`的结果有个清晰的认识。

在没有协议的情况下，好像是参数中不能出现协议或者端口(:号)，否则就不会把第一段解析成host，虽然还不知道为什么，这里暂且不论。

![image.png-151.7kB][9]

在这种情况下，我们只需要把后面可能出现的http去掉就好了，因为无协议的情况下会默认补充http在前面（一般来说）。

2、curl必须要能把空host解析成localhost，所以libcurl版本要求在7.54.0以下，而且目前测试只影响windows服务器（欢迎打脸

3、dz必须在80端口下

在满足上面的所有条件后，我们实际请求了本地的任意目录
```
http://://{可控}

===>

http://127.0.0.1/{可控}
```
但这实际上来说没有什么用，所以我们还需要一个任意url跳转才行，否则只能攻击本地意义就很小了。

# 任意url跳转

为了能够和前面的要求产生联动，我们需要一个get型、不需要登陆的任意url跳转。

dz在logout的时候会从referer参数（非header头参数）中获取值，然后进入301跳转，而这里唯一的要求是对host有一定的验证，让我们来看看代码。

`/source/function/function_core.php:1498`

![image.png-530.1kB][10]

上面的截图解释了这段代码的主要问题，核心代码为红框部分。

为了让referer不改变，我们必须让host只有一个字符，但很显然，如果host只能有一个字符，我们没办法控制任意url跳转。

所以我们需要想办法让`parse_url`和`curl`对同一个url的目标解析不一致，才有可能达到我们的目标。
```
http://localhost#@www.baidu.com/
```

上面这个链接`parse_url`解析出来为`localhost`，而curl解释为`www.baidu.com`

我们抓个包来看看
![image.png-189.6kB][11]

成功绕过了各种限制

# 利用

到现在我们手握ssrf+任意url跳转，我们只需要攻击链连接起来就可以了。攻击流程如下

```
cutimg ssrf link
=====>
服务端访问logout任意url跳转
====301====>
跳转到evil服务器
=====302=====>
任意目标，如gophar、http等
```

当然最开始访问cutimg页面时，需要先获取formhash，而且referer也要相应修改，否则会直接拦截。

exp演示

![image.png-121.5kB][12]


  [1]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/xo5e2sj7ia0m47hcpwva09e6/image.png
  [2]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/bamsf7n2994y5npkltw1w7rf/image.png
  [3]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/421pi5euokvsnvybhmq65yev/image.png
  [4]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/p9eqr2sw7dgptqpduhwid5vu/image.png
  [5]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/gtcehffv4rw4u10ivp5j2odj/image.png
  [6]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/uwznq733p726euelbk47rman/image.png
  [7]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/v24oetlf4jcuhzuvd0cun3qm/image.png
  [8]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/t666l41iig9cad7xxcxju3ti/image.png
  [9]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/0y3l8uvmz3tgme0sbruq6dr0/image.png
  [10]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/ije6y4dpa8hyl787ed3t8xdj/image.png
  [11]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/9n2nixxhzlbq3us1f8xrjzu0/image.png
  [12]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/cge5q53wra5p828wdhzuglgx/image.png
