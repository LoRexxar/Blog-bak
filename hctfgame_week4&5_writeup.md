title: hctf_game_week4&amp;5_writeup
date: 2016-03-14 19:25:35
tags:
- Blogs
- ctf
categories:
- Blogs
---

假期难得有时间空闲下来，就和协会的小伙伴组织了一次比较简单的ctf比赛针对学校的学弟学妹们，这里就贴上每一次的writeup，以供整理复习用。

<!--more-->

# week4

## MISC

### MISC编码xxx	    POINT: 95 DONE 
本题题解详情
题目ID： 70
题目描述： http://ctf.lazysheep.cc:8084/coding.html
Hint: (╯‵□′)╯︵┻━┻ 

打开是一大堆()[]{}，是jsfuck，有兴趣的可以去搜搜看，这样的题目一般都是用chrome的f12或者firefox的firebug，大黑客比较坑，写了while（1）的循环弹窗，没找到什么优雅的解法，我是用firebug跑起来，然后电脑就炸了，过了几秒后firefox会弹窗提示发生了错误，然后调试....大家研究吧，说不定有什么奇怪的解法...


# week5

## pentest

### lightless&aklis的渗透教室-番外篇1	    POINT: 150 DONE 
本题题解详情
题目ID： 72
题目描述： http://120.27.53.238/pentest/ex01/index.html
Hint: 1.天天让你们交WP，你们真的会git了吗？
2.你们git init完了之后没发现当前目录下多了个文件夹？不想知道这文件夹是干啥的？
3. ".git"文件信息泄露


题目对于很多人来说可能真的很难，但是给的hint已经比较清楚了，首先知道题目是git hack，网上会搜到好几个工具，有个githack.py写的比较糟糕，基本什么都爬不到，有个比较成熟的工具是:
[https://github.com/kost/dvcs-ripper](https://github.com/kost/dvcs-ripper)

需要安装个perl环境，windows下安这个简直和吃了屎一样难受，我觉得还是linux下比较简单，所以推荐去linux下安，这样就能爬下来一个完整的.git。

然后你需要稍微了解依稀git命令，git是树状结构，所以每个版本的文件都可以回滚，首先git log看看有什么，发现了之前的版本，然后git reset -HEAD xxxx就可以回滚到指定的版本，xxxx是指版本信息的头....

有兴趣的去尝试一下吧。

### lightless&aklis的渗透教室-番外篇2	    POINT: 50 DONE 
本题题解详情
题目ID： 73
题目描述： http://120.27.53.238/pentest/ex02/index.html
Hint: 1.这题还要hint？Vim你们会么？
2.你在用vim写代码的时候，突然vim崩了，然后呢？
3.vim不正常退出时的swp恢复文件

hint比较清晰了，可以明白是vim崩了会产生的缓存文件，测试一下就知道了，.xxx.swp是vim崩了的时候会产生的备份文件，一般如果崩了，打开时会提示是否恢复....测试一下就知道了。


### lightless&aklis的渗透教室-番外篇3	    POINT: 100 DONE 
本题题解详情
题目ID： 74
题目描述： http://120.27.53.238/pentest/ex03/
Hint: 1.不会做这题只能说基础不扎实，你们真的会日站么？
2.网络爬虫排除标准

hint还是关键，网络爬虫排除标准是指一般有个robots.txt文件，里面会写避免被爬的地址，打开可以看到两个路径，其中一个有个任意文件包含，可以读到real_flag.

### lightless&aklis的渗透教室-番外篇4	    POINT: 50 DONE 
本题题解详情
题目ID： 75
题目描述： http://120.27.53.238/pentest/ex04/
Hint: 1.你真的会VIM了吗？
2.VIM自动备份文件

不知道有没有在番外2中踩了坑，因为可能会搜到vim的备份文件，一般是xxx.php~这样的备份文件，测试下就得到了。

### lightless&aklis的渗透教室-番外篇5	    POINT: 100 DONE 
本题题解详情
题目ID： 77
题目描述： http://120.27.53.238/pentest/ex05/
Hint: 1.这次轮到玩坏email了~
2.很古老的email编码方式，用了两种编码方法。

email的编码方式提示出来基本就清晰很多了，题目是qp+uue decode，但是有一些坑，由于编码问题，网上的很多解码网站都是无效的，这里有个测试有效的网站：
[http://web.chacuo.net/charsetuuencode](http://web.chacuo.net/charsetuuencode)
去试试吧。