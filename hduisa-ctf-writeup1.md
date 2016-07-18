title: Hduisa_ctf_writeup01
date: 2015-02-26 16:23:29
tags:
- Blogs
- ctf
categories:
- Blogs
---
假期一直在玩会长大大搞得ctf集训，一个假期也学到了，现在整理writeup，也对假期学习到的东西有一个清晰的认识...

<!--more-->

# WEEK1_0x01_小明的女神 #

打开题目看到的是一幅女神的图片

![](/img/hduisa_ctf_writeup1/1.jpeg)
看上去很熟悉，很像是之前hctf做过的题目，于是查看源码
![](/img/hduisa_ctf_writeup1/2.jpeg)

把这串代码通过base64图片解码之后得到的是和一样的图片，这里一下就没什么想法了，后来通过问学长得知把两张图片通过**Bcompare**，可以得出这么一串不同的数字。

**74.117.56.51.114.111.67.51**

到这里直接脑洞碎掉了，一直都没什么想法，后来看到别人的writeup，得知把数字转换为acsii码得到Ju83roC3...

然后加在题目链接后面成为：
- **http://104.236.171.163/week1/nvshen/Ju83roC3/**

Flag get！

ps：这里脑洞简直碎了一地...

# WEEK2_0x02_getresponse #

第一眼看到这道题的时候，我简直被吓哭了...毫无思路，提示告诉我们是来自去年alictf的一道题目，关键在于这篇文章**http://drops.wooyun.org/tips/750**，一下子可以想到这道题是要使用@的黑魔法...但是试了好多毫无思路，最后看了writeup我的人生观被颠覆了，得到关键点在于help下的提示
![](/img/hduisa_ctf_writeup1/3.jpeg)
![](/img/hduisa_ctf_writeup1/4.jpeg)

之前完全没有注意到，这里有两个字符在不断的刷新下会变，多次刷新后得到

**R2V0IFJlc3BvbnNlIEZyb20gaGR1aXNhLmNu**

解密出来得**Get Response From hduisa.cn**

一下子得到方向构造**http://drops.hduisa.cn@hduisa.cn**
![](/img/hduisa_ctf_writeup1/5.jpeg)

Flag get！

# WEEK1_0x03_easy sqli #

开始看到看到这一题的时候真的是毫无思路，因为毕竟还没有开始学习sql和php，于是看到了之后的writeup，也算是对sql的小入门吧...

题目是一道很有意思的题目
![](/img/hduisa_ctf_writeup1/6.jpeg)

看到mq==第一反应是base64的编码，也就是1...之后发现题目有各种坑，会长及时给出了提示...
![](/img/hduisa_ctf_writeup1/7.jpeg)

首先就是坑1，如果没有修改传递方式的话，无论提交什么，都会
![](/img/hduisa_ctf_writeup1/8.jpeg)

再加上坑3，过虑了and和or于是构造出**'||1#** base64编码后得到**J3x8MSM=**添加到登录表单上，并且修改提交方式
![](/img/hduisa_ctf_writeup1/9.jpeg)

提交，Flag get！
![](/img/hduisa_ctf_writeup1/10.jpeg)


总体来说WEEK1的题目难度和综合程度都比较高，于是我等菜鸟一道也没做出来，不过还是不错，学到很多东西...

# WEEK2_0x01_颇有技巧的sql注入 #

这道题实在是难度太大，属于答案给我我都做不出的那种，这里还是放上别人的writeup仅供参阅

- http://lazysheep.cc/2015/02/01/0x0F/

- http://www.edwardl.xyz/2015/02/06/hduisa_ctf_week2_writeup/



# WEEK2_0x02_最简单的题目 #

因为上周的题目难度过大，这周给出了许多简单的题目，这道题的也属于传说中做不出不要玩ctf的类型.

后来想到这题，一直感觉是被会长耍了，题目说要get id=flag，谁知道直接查看源码就可得知
![](/img/hduisa_ctf_writeup1/11.jpeg)

Flag get！这题太多简单，也就不赘述...

# WEEK2_0x03_基础训练 #

题目比较简单，而且是比较有意思的那种，题目提示告诉我们是关于cookie和编码的
![](/img/hduisa_ctf_writeup1/12.jpeg)

根据页面上面的提示，图片搜索这个人，得到罗马音为**Tobiichi Origami**

根据题目中的提示，查看cookie
![](/img/hduisa_ctf_writeup1/13.jpeg)

发现**flag:bGlnaHRsZXNzOjQ3NmExMGYxZjNkZGU1MWQxYjUyMjY3YzFhY2NjMzll**

把后面的东西用base64解码后得到

**lightless:476a10f1f3dde51d1b52267c1accc39e**

开始做到这里时候有点儿卡住了，突然想到可能是md5，用md5解码后得到，后面那串是lightless，一下子明白了

把上面的的罗马音照格式修改出来，修改cookie，提交...（这里使用的是firefox的temper data）
![](/img/hduisa_ctf_writeup1/14.jpeg)


Flag get！
