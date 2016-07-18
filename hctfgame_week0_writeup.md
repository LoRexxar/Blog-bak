title: hctf_game_week0_writeup
date: 2016-02-18 9:25:35
tags:
- Blogs
- ctf
categories:
- Blogs
---

假期难得有时间空闲下来，就和协会的小伙伴组织了一次比较简单的ctf比赛针对学校的学弟学妹们，这里就贴上每一次的writeup，以供整理复习用。

<!--more-->

## WEB

###  WEB从0开始之0     POINT: 10 DONE

题目ID： 26
题目描述： WEB页面的HTML，CSS，JS客户端是可以查看的哦～你能在平台源码中找到FLAG么？
Hint: 不知你有没有发现，通过右键看到的源码中没有题目，没有排名信息。
推荐：chrome -> F12; firefox -> firebug

题目描述我觉得说的很清楚了，就是要从源码中找到flag，而之所以提示中说道右键源码中没有题目，没有排名信息，是由于那一部分使用了ajax，有兴趣的可以去尝试下。

###  WEB从0开始之0.1     POINT: 20 DONE

题目ID： 27
题目描述： 你知道一个网页从输入URL到显示出页面，都经历了啥么？
http://ctf.lazysheep.cc:8080/
Hint: Do you know HTTP headers?

这题题目是在hctf2015中的签到题，大概就是点进去有个302的跳转，从index.php跳到index.html，有很多中办法可以做到，包括抓包，可以用temper data或者burp，f12应该也是可以看到的，还有一种就是crul -I命令，也可以看到，很简单就不赘述了。

###  WEB从0开始之0.2     POINT: 20 DONE

题目ID： 35
题目描述： 你知道啥是cookie吗？
那么你会修改它吗？
http://ctf.lazysheep.cc:8080/web0-2.php

题目描述比较明白了，就是说需要修改cookie，同样工具比较重要，一般使用chrome -> F12; firefox -> firebug，抓包改包当然也可以，修改为true就好了

## MISC

###  MISC从0开始之编码0     POINT: 10 DONE

题目ID： 25
题目描述： SENURntUSElTSVNCQVNFNjRFTkNPREV9
Hint: base系列编码

题目主要是要大家熟悉base64编码，看到全大写本以为是base32，结果还是简单的base64编码，解码方式很多种，站长工具，或者手头的工具应该都很随意...

###  MISC从0开始之Steganography0     POINT: 10 DONE

题目ID： 32
题目描述： AK菊苣的小姐姐们之0～
http://ctf.lazysheep.cc:8080/steg0.html

最最基础的图片题目，这里引出一个比较重要的图片处理工具，叫做stegslove，功能非常强大，直接使用查看图片信息即可。

###  MISC从0开始之流量分析0     POINT: 10 DONE

题目ID： 33
题目描述： http://ctf.lazysheep.cc:8080/net0.pcap
Hint: PS: FLAG打错了。。格式变成flag{}..懒得改了

最最基础的流量分析题，基本上来说，分析流量使用的都是wireshark这个，在对注册页面的一个http请求处，可以看到一个明文的flag请求，get！

###  CTF coding step0     POINT: 50 DONE

题目ID： 30
题目描述： 打CTF就是拿工具？ 不不不，也要写很多代码的。这个系列就是让你熟悉CTF风格的编程题目，具体的要求见题目吧のの 就是让你们多看点英文：
nc 115.29.77.78 9999
Hint: 用telnet或者nc连接如上地址和端口，windows下没有的请自行寻找ssh/telnet工具

题目比较简单，即便是手输很多遍A都是可以的，主要是熟悉nc，有兴趣可以去查查。

## crypto

### 密码学从0开始之0     POINT: 10 DONE

题目ID： 23
题目描述： ojam{AopzpzJhlzhyWhzzdvyk}
Hint: Caesar's code

凯撒密码，比较简单，大概写个脚本就好了，c也同样可以实现。

## pen test

###  lightless的渗透教室-1     POINT: 50 DONE

题目ID： 28
题目描述： http://120.27.53.238/pentest/01/http-method.php
提交flag时，请连同hctf{}花括号一起提交。
Hint: 暂无HINT

认真看文档哟！！



