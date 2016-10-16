title: gif bypass CSP?
date: 2016-04-20 19:48:33
tags:
- Blogs
- csp
categories:
- Blogs
---
前段时间看到了一个有趣的bypasscsp的文章，最开始是在html5sec上看到的
[http://html5sec.org/#138](http://html5sec.org/#138)
这里本来说的是关于link的import属性，但示例中却使用gif bypass了csp，研究了一下午，发现了一些有趣的东西。
<!--more-->

# 原文
首先是原文
[http://html5sec.org/cspbypass/](http://html5sec.org/cspbypass/)
我们看到作者的标题是**CSP Bypass in Chrome Canary + AngularJS**
并且如果你使用了chrome浏览器(值得注意的问题是这个demo只有chrome才会弹窗)，可以明显的看到有弹窗。

那么让我们来分析一下demo吧

## 原理
这个方式的原理来自于[http://www.slideshare.net/x00mario/jsmvcomfg-to-sternly-look-at-javascript-mvc-and-templating-frameworks/48](http://www.slideshare.net/x00mario/jsmvcomfg-to-sternly-look-at-javascript-mvc-and-templating-frameworks/48)

![](/img/ccsp/1.png)
首先通过上传带有信息的gif图片,让gif与站在同源环境下
```
GIF89ad=1/*xxxxxx*/;alert(1)/*<script src="test.gif"></script>,xxxxxxxxxxxxxxxxxxxxxxxxxxxxx<link rel="import" href="test.gif" />*/
```
构造class=ng-include:"test.gif"来引用test.gif,angularjs会把gif的内容解析到页面内。
```
<span style="visibility:hidden" class="ng-include:'test.gif'"></span>
```
会变成
```
<span style="visibility:hidden" class="ng-include:'test.gif' ng-scope">

<span class="ng-scope">
GIF89ad=1/*xxxxxxxxx*/;alert(1)/*
</span>
<script src="test.gif" class="ng-scope"></script>
<span class="ng-scope">,xxxxxxxxxxxxxx;
</span>
<link rel="import" href="test.gif" class="ng-scope">
<span class="ng-scope">*/
</span>
</span>
```
我们能看到3个请求
![](/img/ccsp/2.png)
但是我们也能看到有部分被CSP拦了
![](/img/ccsp/3.png)
根据这个ppt中的解释来说，是`<link rel="import" href="test.gif" class="ng-scope">`加载了test.gif,我们看看这个link标签里面又是什么呢。
```
<html>
<head>
</head>
<body>
GIF89ad=1/*xxxxx*/;alert(1)/*<script src="test.gif"></script>,xxxxxxxx;<link rel="import" href="test.gif">*/</body>
</html>
```
![](/img/ccsp/4.png)
link中又加载了一次test.gif

这里成功执行了`<script src="test.gif"></script>`
成功弹窗。

## 真的就这么简单吗？
看上原理就如同所述的那样，但是在我的测试下实际情况和demo中有一切区别
[demo](http://html5sec.org/cspbypass/)
[我的测试环境](http://119.29.192.14/test.php)
我们发现一切都是熟悉的，但是原本的那条会导致弹窗的`<link>`出现了一条报错
```
Refused to execute script from 'http://119.29.192.14/test.gif' because its MIME type ('image/gif') is not executable.
```
查了一下发现这里的报错大多都是由于**X-Content-Type-Options**这个头造成的，他通过查看响应中的content-type是不是与预期相符判断的，这里传入的test.gif MIME type为image/gif，和预期的js不符，所以被拒绝了，具体可以看
[http://drops.wooyun.org/tips/1166](http://drops.wooyun.org/tips/1166)
[https://blogs.msdn.microsoft.com/ie/2008/07/02/ie8-security-part-v-comprehensive-protection/](https://blogs.msdn.microsoft.com/ie/2008/07/02/ie8-security-part-v-comprehensive-protection/)

但是我们很明显的发现，这个头属性并没有出现在response的请求头中，但事实就是这个属性应该是默认开启的，那么我们能修改图片respense的context-type属性?

我们可以看到在demo和我的测试环境中关于test.gif的请求明显是不同的的。
![](/img/ccsp/5.png)
![](/img/ccsp/6.png)

在请求头中我们明显看到有两个特殊的地方。
1、Content-Type
显示此HTTP请求提交的内容类型。一般只有post提交时才需要设置该属性。
[http://tool.oschina.net/commons](http://tool.oschina.net/commons)

2、Content-Location:test.gif.js
请求资源可替代的备用的另一地址
也就是如果test.gif没有请求到，那么久使用test.gif.js....那么这个设置到底是干嘛的...

### content-location:test.gif.js?
如果我们将script中的src改为test.gif.js，我们看到请求变了
![](/img/ccsp/7.png)

我们发现刚才的报错消失了，但这样一来，如果能够在同源环境下上传一个.js后缀，那么所谓的bypass csp也就没有意义了。

### content-type
在服务器的配置中，可以通过修改配置文件将.gif的默认context-type改为js，这样以来，所谓的.gif也就是一个.js了，从本质上来说并没有什么区别。

那么这个洞到底存不存在呢