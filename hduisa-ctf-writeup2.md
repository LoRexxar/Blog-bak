title: Hduisa_ctf_writeup02
date: 2015-02-26 16:43:43
tags:
- Blogs
- ctf
categories:
- Blogs
---
# WEEK3_0x01_做不出来不要玩CTF系列之二 #

<!--more-->
这道题算是比较基础的题目算是综合性比较强的简单题目，题目的大意是说flag分为3段，涉及到编码跳转以及信息收集能力，所以是一道比较有意思的题目。

第一段flag，是一段非常容易忽略的flag，但是仔细寻找后可以找到，在响应头信息里，可以找到这一部分flag...
![](/img/hduisa_ctf_writeup2/1.jpeg)

第二段flag非常的有意思，仔细观察初始页面的源代码之后，可以非常清晰的发现页面在点击后会发生跳转...

![](/img/hduisa_ctf_writeup2/2.jpeg)
![](/img/hduisa_ctf_writeup2/3.jpeg)


于是这里想办法阻止这个跳转，我使用的仍然是firefox的temper data插件，阻止跳转后可以看到一个白色的画面，发现会长非常猥琐的把字体设置成了白色...

![](/img/hduisa_ctf_writeup2/4.jpeg)

Get flag2!

这里还有两种相当简单的办法，提供在没用工具时阻止网页跳转。
1.改变internet设置，在安全-->自定义设置把META TEFRESH禁用，就可以阻止跳转。
![](/img/hduisa_ctf_writeup2/5.png)

2.如果你觉得上面的办法都太麻烦，就手动阻止吧，眼疾手快就好.

第三段flag比较特殊，是一串关于（【{的特殊符号，开始看到我直接懵了，后来问过学长之后得知把这串字符拖去百度，就可以得到答案.
![](/img/hduisa_ctf_writeup2/6.png)

也就是这个网站**http://www.cnblogs.com/pandora/archive/2010/02/27/1674833.html**

不得不承认这种方式的天才之处

复制粘贴，get flag3！

![](/img/hduisa_ctf_writeup2/7.png)

Flag get！

# WEEK3_0x02_SQLI系列一 #

# WEEK3_0x03_SQLI系列二 #

上面的两道题又是属于sql的系列，无奈放弃，这里贴上别人的writeup...

**http://lazysheep.cc/2015/02/07/0x11/**

# WEEK3_0x04_做不出来不要玩ctf之三 #

这道题不得不说，绝对是会长埋下的坑，打开看到提示，得知是get id=？，初开始以为是注入，但是开始一个个尝试.
![](/img/hduisa_ctf_writeup2/8.png)

试到3的时候不再变化，于是根据提示实验各种abc的组合，没有发现任何变化，于是继续寻找提示，继续get，到id=6时出现了flag，一下子晃下了我的狗眼 ...233333

![](/img/hduisa_ctf_writeup2/9.png)
Flag get!


# WEEK4_0x01_baD #

这周的题目总体来说算是比较难的，第一题打开，提示告诉我们vim的提示

![](/img/hduisa_ctf_writeup2/10.png)
由于实在没有接触过关于vim的东西，所以百度了很久都没找找到什么有用的线索，只知道vim是linux环境下很强大的编辑器，询问学长后，学长给出的提示是备份文件，百度还是未果，但是google可以得到一个很有效的提示...

![](/img/hduisa_ctf_writeup2/11.png)

得知vim会自动保存swp后缀的备份文件，这里需要注意的是文件的名称是.xxx.php.swp

成功下载到源文件，查看源代码之后得到flag。

Flag get!

# WEEK_0x02_white #

这道题的提示是xss，但是后来得到会长的提示，告诉我们这题还是一道注入，无奈，只能放弃..

这里贴上别人的writeup。

**http://lazysheep.cc/2015/02/15/0x13/**

# WEEK_0x03_懒得起名字了 #

题目非常的有意思，是一个登录面板，四处查看后，在响应头信息里发现提示.bak

![](/img/hduisa_ctf_writeup2/12.png) 

下载备份文件后可以得到源代码。

![](/img/hduisa_ctf_writeup2/13.png)


源代码中很清晰的提示到，用户名是lightless，密码是abc!10O0o，

但如果用户名是lightless会爆出your account is blocked!

存在矛盾，无法登陆，而且输入经过编码，无法注入，所以办法绕过验证。

当时想到的办法是特殊符号，使得sql无法识别蛋php可以识别，后来证明想法没错，但是有点儿简单。

看过会长的writeup之后，得知有两种办法，

- 第一种为改变大小写，相当的简单，一下子就可以得到flag。

- 第二种和之前的想法类似，但是要用fuzz尝试特殊符号，符合的符号还是相当多的。

![](/img/hduisa_ctf_writeup2/14.png)


最后，Flag get!

 

总体来说，整个假期的ctf的体验还是学习到了相当多的东西，也清楚的认识到现在学习php的进度应该加快，才有机会得到更多的东西。加油！
