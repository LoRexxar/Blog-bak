---
title: pwnhub_迷
date: 2016-12-26 23:21:49
tags:
- pwnhub
- fastcgi
- ssh_sock5
categories:
- Blogs
---

这是一个从头到尾都让我觉得很迷的题目┑(￣Д ￣)┍

<!--more-->

http://54.223.142.7:1234/

什么东西都没有，扫端口找到了一些信息
```
root@iZ285ei82c1Z:~# nmap -sT -Pn 54.223.142.7 -p1-65535

Starting Nmap 6.40 ( http://nmap.org ) at 2016-12-23 21:19 CST
Nmap scan report for ec2-54-223-142-7.cn-north-1.compute.amazonaws.com.cn (54.223.142.7)
Host is up (0.027s latency).
Not shown: 65521 closed ports
PORT     STATE    SERVICE
22/tcp   open     ssh
80/tcp   open     http
135/tcp  filtered msrpc
136/tcp  filtered profile
137/tcp  filtered netbios-ns
138/tcp  filtered netbios-dgm
139/tcp  filtered netbios-ssn
445/tcp  filtered microsoft-ds
593/tcp  filtered http-rpc-epmap
1234/tcp open     hotline
4444/tcp filtered krb524
5554/tcp filtered sgi-esphttp
6176/tcp filtered unknown
9996/tcp filtered unknown

```

1234和80分别是apache和nginx....

扫目录找到README.md
```
Loli Club Proxy Server
=====

## Apply
发送邮件到 baka@loli.club 申请

## Usage
ssh proxy@54.223.142.7
```

现在已知信息：
1、ssh proxy@54.223.142.7这个禁止登陆，密码是proxy，证明有一个代理的服务在。
2、openssh7.2p2，不存在cve读取文件

这里是个不懂的黑科技，叫做ssh sock5代理。

具体看这篇文章
[http://os.51cto.com/art/201211/364512.htm](http://os.51cto.com/art/201211/364512.htm)

通过这个命令，可以把远程代理到本地来
```
ssh -qTfnN -D 7070 -p 22 user@host
```

到本地了就和别的代理方式没什么区别了，设置代理然后nmap跑一发。

![image_1b4rjcvh4ua51ake1e5j1eft9rv9.png-88.7kB][1]


  发现本地开启了9000，也就是fastcgi，fastcgi有个远程命令执行，网上有现成的exp，可以试试看，有个比较关键的问题是需要有个php文件。
  
  很好，这次我也不知道该去哪找个php文件了...看别人的wp是去php的拓展插件里，在我的系统里，这部分在这里
  ![image_1b4rjojas1glag04v3m1vpg1bj7m.png-9.7kB][2]

但在wp中这部分路径在/php/share/...

成功执行命令很快就拿到flag了，flag就在`/flag`  



  [1]: http://static.zybuluo.com/LoRexxar/0yzymerrijoi97gvkmg6y199/image_1b4rjcvh4ua51ake1e5j1eft9rv9.png
  [2]: http://static.zybuluo.com/LoRexxar/83hz6jc1zwa3f7e1srj9int9/image_1b4rjojas1glag04v3m1vpg1bj7m.png