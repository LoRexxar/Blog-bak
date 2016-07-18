title: ldpa简单注入
date: 2016-02-28 14:54:26
tags:
- Blogs
- injection
- ctf
---

上个假期快结束的时候应该是打了一个zctf，因为快考试了，所以就接触了一个应该是web2的题目，是一个ldpa注入，以前没有见过，还是整理下学习一下。

<!--more-->

题目是一个登陆框，稍微测试了一下发现没有找到什么注入点，在cookie里发现提示，说不是sql注入，扫下端口发现存在389端口，是ldap端口。
去wooyun上找到一篇文章看看：
[http://drops.wooyun.org/tips/967](http://drops.wooyun.org/tips/967)

稍微fuzz一下发现一共就过滤了一个&，于是在文章中看到的admin)(&))无效，再试试别的。

发现可以用admin/*绕过过滤，这里是因为在ldpa中星号可以替换过滤器中的一个或多个字符。

登陆成功后发现是一个搜索，稍微测试下搜索a回显,0 admin, (| (uid=*a*))，这里还有个提示是说can you find my description，这里description是表名。如果搜索正确就会有回显，于是构造语句盲注。

`test)(description=z`
发现成功回显，那么就简单了，写脚本或者用burp都可以快速解决。get flag!



