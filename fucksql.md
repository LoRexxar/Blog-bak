title: Hduisa_ctf_wee4_fucksql
date: 2015-05-21 11:40:49
tags:
- ctf
- Blogs
categories:
- Blogs
---
这周会长出了两道关于渗透的题目，综合性相当强，几乎已经是一道正式的ctf的200分题目了，题目很简单是这样的。

<!--more-->

## 0x01   FuckMySQL1 ##

**Hint:是渗透题！这是渗透题！虽然也有web的成分！但是这是渗透题！！！**
题目描述：好久没出题了，于是今天出了个渗透题丢出来，最终的奖励就是vps的root shell哦~http://104.236.139.236/week4/fuckmysql/
 

打开给出的页面，出现的是一个图片库，翻了翻没什么的地方，既然描述说是渗透题，那么就先扫描下目录，很快就得到了一大堆奇怪的，首先是index.php.bak，本以为会出现源码，打开看到一屏幕的葫芦娃...666666

www.zip下载下来也是一样的东西，甚至还得到了wojiushihoutai.php这种东西...

最后发现robot.txt下其中一个是有用的，是admina,php，是真正的后台地址。

简单尝试下发现只有账户输入lightless时候会提示密码错误，后面一下子就卡住了，找了找貌似并没有什么提示，而且也不存在注入漏洞...后来问过学长后，得到这个地方是要社工的，比较纠结的是，只有一个社工裤能得到这串密码，做题的时候刚好崩了，所以过了一天才得到密码...

登录成功后进入hhh.php页面，页面中出现好几个提示

![](/img/fucksql/1.jpeg)


第一个提示摆明了就是vim下写了网页，所以进入cat.php后尝试下载缓存文件，cat.php~成功拿到了网页源码...分析源码，得到很重要的信息。

1.需要post key1和 key2 给后台。

2.post 进入数据库的key1查询返回行中key2等于所post的key2时，返回flag

3.post 进入的数据过滤了大部分语句，（），但仍旧存在注入漏洞。

由于对sql和sql注入并没有学习，所以一下子卡住了...

后来经过学长的提醒，这个地方使用了groupby的黑魔法，去百度了两天并没有什么结果，于是去看mysql的官方文档。

**http://dev.mysql.com/doc/refman/5.7/en/group-by-modifiers.html**

得到很重要的信息就是groupby with rollup会出现NULL的值，听从学长的话本地搭建后，尝试构造查询语句。

![](/img/fucksql/2.jpeg)

后面的LIMIT 1 OFFSET 1 可以返回1行后的1行，这样我们就得到了唯一行，并使key2为NULL

后面一下子彻底没了办法，由于对sql注入一点儿办法都没有，所以这里是学长直接给我一个payload...

key1='= '' GROUP BY key2 WITH ROLLUP LIMIT 1 OFFSET 1%23&key2=
这里的'=''并不知道什么意思，后面的%23是为了注释掉后面的额' ，但是了解太少，不知道为什么不能使用--...

![](/img/fucksql/3.jpeg)


这样就拿到了flag，由于最后的sql注入完全是靠学长，所以这个flag暂时不提交了...

## 0x02   FuckMySQL2 ##
上道题给出的一串字符，用base64解下是乱码，观察下，全部都是大写，所以打开py解base32，得到：

'Ni Xiang Yao Shell Me? GoGoGo! Ji Ran Shi Tu Pian Ku, Na Me Tu Pian Ken Ding You Yong!'
