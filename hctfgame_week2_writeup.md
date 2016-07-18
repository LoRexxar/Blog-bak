title: hctf_game_week2_writeup
date: 2016-02-18 11:25:35
tags:
- Blogs
- ctf
categories:
- Blogs
---

假期难得有时间空闲下来，就和协会的小伙伴组织了一次比较简单的ctf比赛针对学校的学弟学妹们，这里就贴上每一次的writeup，以供整理复习用。

<!--more-->

## WEB

###   WEB从0开始之xss challenge0.5     POINT: 20 DONE

题目ID： 56
题目描述： http://115.28.78.16/xss/xss0.5/index.php
1、成功执行prompt(1).
2、payload不需要用户交互
3、payload必须对下述浏览器有效： Chrome(最新版) - Firefox(最新版)
4、将有效payload发送给LoRexxar或者大飞客获取flag
Hint: 暂无HINT

就是个简单的xss入门题目，仅仅过滤了script，而且只是一次过滤，有一万种方法让他弹窗。

`<scrscriptipt>prompt(1)</scrscriptipt>`
`<svg/onload=prompt(1)`
`<img src onerror=prompt(1)`

这些都可以

###  WEB从0开始之xss challenge1     POINT: 100 DONE

题目ID： 57
题目描述： http://115.28.78.16/xss/xss1/index.php
1、成功执行prompt(1).
2、payload不需要用户交互
3、payload必须对下述浏览器有效： Chrome(最新版) - Firefox(最新版)
4、将有效payload发送给LoRexxar或者大飞客获取flag
Hint: 暂无HINT

题目很简单，但是我估计80%的人都不知道是什么原理，先给出两种payload
`<svg><script>prompt&#40;1)</script>`
`<script>prompt反引号1反引号</script>`(妹的我发现md语法这个反引号有用打不出来)

这里的svg不能去掉，由于xml编码特性。在SVG向量里面的script元素（或者其他CDATA元素 ），会先进行xml解析。因此&#x28（十六进制）或者&#40（十进制）或者&lpar；（html实体编码）会被还原成（。

## misc

###  MISC 驾驶技术科目三     POINT: 200 DONE

题目ID： 38
题目描述： 科目三开始路考了啊，小伙子们快上车！ 车票在0x02.jpg
Hint: 无

科目三的入口比较坑爹，开始一直没找到，后来发现是一串神秘代码/s/xxx的网盘地址。。在0x02中就可以找到。

科目三的难度有点儿虚高，我使用的diff -a命令，一下就能看到一个base64编码过的差异，解码再顺着找就可以get了。

###  MISC 驾驶技术科目四     POINT: 250 DONE

题目ID： 39
题目描述： 开五档上高速，教练带你秋名山。

科目四的难度就比较高了，这里是xdctf的misc2，压缩包明文攻击。
这里找到一个win环境下非常好用的软件Advanced Archive Password Recovery
大概20分钟就能跑出来了。

## pentest

###  lightless&aklis的渗透教室-3     POINT: 50 DONE

题目ID： 61
题目描述： http://120.27.53.238/pentest/03/http-cookie.php

注意看文档！！还有这种题目一般要抓包翻翻看，就能得到提示了。。。
