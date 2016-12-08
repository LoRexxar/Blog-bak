---
title: using polyglot JPEGs bypass CSP 分析
date: 2016-12-07 21:30:20
tags:
- Blogs
- csp
categories:
- Blogs
---

前段时间，外国爆出来的新的bypass CSP方式，这里稍微研究下。

[http://blog.portswigger.net/2016/12/bypassing-csp-using-polyglot-jpegs.html](http://blog.portswigger.net/2016/12/bypassing-csp-using-polyglot-jpegs.html)

但实际检查整个逻辑之后，我觉得应该算作是对上传检查的绕过，不能算作是bypass csp

<!--more-->

文章里提到通过创建一个多语言的JavaScript/JPEG 来绕过CSP，他给了两张demo图片，回顾整个逻辑。

我们打开给出的demo图片和随便一张jpg图片，首先前四位是JPEG头`0xFF 0xD8 0xFF 0xE0`，如果你曾尝试过把一张图片当作script来执行的话，应该会知道在js的逻辑里，执行到错误的位置位置，如果出现乱码无法执行，那么就会直接报错不执行。

所以在demo图片中，头后跟着`0x2F 0x2A` 也就是`\*`，这里的解决办法是注释，但这两位本身其实是头部的长度，所以在demo图片中用大量的**00**填充空白，紧接着在最后段加上`0x2A 0x2F`结束注释，后面跟上js的payload。

最后呢，要处理后面大段图片，我们还要加上注释，正常的文件尾是`0xFF 0xD9`，在前面加上注释结束`2A 2F 2F 2F`，很稳健。

这样就形成了一个完整的图片js，既有图片的结构，又有js的代码，有一个特别的问题就是，如果用utf-8作为编码时，包含脚本代码时，会把整个语义打乱，所以这里需要指定编码为 ISO-8859-1。

payload:
```
<script charset="ISO-8859-1" src="http://portswigger-labs.net/polyglot/jpeg/xss.jpg"></script>  
```

比较有趣的一点是，这里chrome拦截了这部分，会爆出

```
Refused to execute script from 'http://portswigger-labs.net/polyglot/jpeg/xss.jpg' because its MIME type ('image/jpeg') is not executable.
```

而在`Safari, Firefox, Edge and IE11`中成功执行了。

但值得思考的是，这里事实上并不能算作是绕过了CSP，因为这里的CSP为

```
Content-Security-Policy: script-src 'self' 'unsafe-inline'
```

所以图片仍然必须为站内，所以事实上，这里其实算作是绕过了站内的图片上传判断Orz