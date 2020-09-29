---
title: pwnhub 之小m的复仇
date: 2017-03-27 00:04:39
tags:
- pwnhub
- xss
categories:
- Blogs
---


做题的时候思路差不多是对的，但是没想明白，讲道理是菜了，稍微整理下，这是一个比较特别的利用方式。

<!--more-->

纵观整个站，看着很大但是有用的功能并不多，看上去唯一的输入点是注册的第三个参数，但是没意义，会经过实体编码。所以我感觉关键在于怎么找到xss点。

1、首先，classes.php这里有专门的路由，而且这个页面里有个单独存在的css，`http://52.80.19.55/classes.php/`，这里肯定是故意的。

2、css的引入方式是相对路径，`<link rel="stylesheet" type="text/css" href="../../classes.css"> `，最开始我猜测是能通过某种方式引入外部的css，css中可以再搞事情xss，但事实证明，这里和我测试过的很多站一样，无论多少个点都去不掉目标最后的斜杠，而存在斜杠的的域是不能通过@来跳到别的地方的，所以必须要有别的利用方式或者站内有可控的点。

3、结合上条，回忆页面内的可控点，只有user.php页面内一部分，是通过注册的第三个参数控制的，上面提到了，这里没有xss点。

4、report bug里面关于url的检测是要以`http://52.80.19.55/`为开始，斜杠必须存在，不能绕过，所以只有传入域内的url


为了连接上面的各个信息，有了一种思路，通过特殊的请求classes.php，使css请求user.php，通过修改user.php页面内的内容，构造任意的css

事实就是...思路是对的，但这里有坑...

正确的payload
```
http://52.80.19.55/user.php/257/0/1/..%2f..%2f..%2f..%2fclasses.php
```

这是一个浏览器和服务端的不同解析方式问题，对于服务端来过，经过一次urldecode
```
上面的目录变成了http://52.80.19.55/classes.php
```

但静态资源不一样，静态资源是浏览器请求的，而浏览器认为..%2f..%2f..%2f..%2fclasses.php是一个目录。

所以我当时天真的用burp修改请求目录是错的...

![image_1bc5l380mpcj1737dtq1nol1c6u9.png-72.8kB][1]

剩下的就是注册个个人信息页面，设置背景图片为自己的服务器，就可以收到flag了

全题的思路近似于
[http://blog.innerht.ml/cross-origin-css-attacks-revisited-feat-utf-16/](http://blog.innerht.ml/cross-origin-css-attacks-revisited-feat-utf-16/)

![image_1bc5llrt0k75jbmai41tas1jo416.png-48.5kB][2]


  [1]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/7phtdg0ehxomaqdkem4hd6j9/image_1bc5l380mpcj1737dtq1nol1c6u9.png
  [2]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/9c3xlhhqb0fjqedmd544o27u/image_1bc5llrt0k75jbmai41tas1jo416.png