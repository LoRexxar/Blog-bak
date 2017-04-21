---
title: Shadow Brokers大新闻整理
date: 2017-04-21 16:18:00
tags:
- Shadow Brokers
- 0day
- windows

---


上周末大半夜的突然爆了大新闻，Shadow Brokers公布了一批美国国家安全局所使用的黑客工具，里面有很多windows的攻击工具，甚至通杀win10和最近版的windows server

[http://www.freebuf.com/news/131994.html](http://www.freebuf.com/news/131994.html)

[https://github.com/x0rz/EQGRP_Lost_in_Translation](https://github.com/x0rz/EQGRP_Lost_in_Translation)

[https://github.com/misterch0c/shadowbroker](https://github.com/misterch0c/shadowbroker)

<!--more-->

# 概况 #

解压完主要有几个文件夹，Windows, Swift和OddJob。

Windows文件夹中包含众多针对旧版Windows操作系统的黑客工具，影响的范围包括Windows XP和Server 2003，通过通过SMB和NBT甚至可以通杀最近版win server和win10。

OddJob能够运行在Windows Server 2003 Enterprise到Windows XP专业版的系统上。文件夹中包含一个基于Windows的植入软件并且包含一些配置文件和payload，听说是可以通杀所有主流杀毒软件的rootkit。

SWIFT的文件夹，其中包含了NSA对SWIFT银行系统发动攻击的相关证据。对我们来说不是太重要。

现在已经被人挖掘并整理的有下面这些

## Exploits ##

- **EARLYSHOVEL** RedHat 7.0 - 7.1 Sendmail 8.11.x exploit

- **EBBISLAND (EBBSHAVE)** root RCE via RPC XDR overflow in Solaris 6, 7, 8, 9 & 10 (possibly newer) both SPARC and x86.

- **ECHOWRECKER** remote Samba 3.0.x Linux exploit. 

- **EASYBEE** appears to be an MDaemon email server vulnerability

- **EASYFUN** EasyFun 2.2.0 Exploit for WDaemon / IIS MDaemon/WorldClient pre 9.5.6

- **EASYPI** is an IBM Lotus Notes exploit  that gets detected as Stuxnet 

- **EWOKFRENZY** is an exploit for IBM Lotus Domino 6.5.4 & 7.0.2

- **EXPLODINGCAN** is an IIS 6.0 exploit that creates a remote backdoor

- **ETERNALROMANCE** is a SMB1 exploit over TCP port 445 which targets XP, 2003, Vista, 7, Windows 8, 2008, 2008 R2, and gives SYSTEM privileges (MS17-010)

- **EDUCATEDSCHOLAR** is a SMB exploit (MS09-050)

- **EMERALDTHREAD** is a SMB exploit for Windows XP and Server 2003 (MS10-061)

- **EMPHASISMINE** is a remote IMAP exploit for IBM Lotus Domino 6.6.4 to 8.5.2

- **ENGLISHMANSDENTIST** sets Outlook Exchange WebAccess rules to trigger executable code on the client's side to send an email to other users

- **EPICHERO** 0-day exploit (RCE) for Avaya Call Server

- **ERRATICGOPHER** is a SMBv1 exploit targeting Windows XP and Server 2003 

- **ETERNALSYNERGY** is a SMBv3 remote code execution flaw  for Windows 8 and Server 2012 SP0 (MS17-010)

- **ETERNALBLUE is** a SMBv2 exploit for Windows 7 SP1 (MS17-010)

- **ETERNALCHAMPION** is a SMBv1 exploit

- **ESKIMOROLL** is a Kerberos exploit targeting 2000, 2003, 2008 and 2008 R2 domain controllers

- **ESTEEMAUDIT** is an RDP exploit and backdoor for Windows Server 2003

- **ECLIPSEDWING** is an RCE exploit for the Server service in Windows Server 2008 and later (MS08-067)

- **ETRE** is an exploit for IMail 8.10 to 8.22 

- **ETCETERABLUE** is an exploit for IMail 7.04 to 8.05

- **FUZZBUNCH** is an exploit framework, similar to MetaSploit

- **ODDJOB** is an implant builder and C&C server that can deliver exploits for Windows 2000 and later, also not detected by any AV vendors 

- **EXPIREDPAYCHECK** IIS6 exploit

- **EAGERLEVER** NBT/SMB exploit for Windows NT4.0, 2000, XP SP1 & SP2, 2003 SP1 & Base Release

- **EASYFUN** WordClient / IIS6.0 exploit

- **ESSAYKEYNOTE** 

- **EVADEFRED**


## Utilities ##

- **PASSFREELY** utility which "Bypasses authentication for Oracle servers"

- **SMBTOUCH** check if the target is vulnerable to samba exploits like ETERNALSYNERGY, ETERNALBLUE, ETERNALROMANCE 

- **ERRATICGOPHERTOUCH**  Check if the target is running some RPC

- **IISTOUCH** check if the running IIS version is vulnerable

- **RPCOUTCH** get info about windows via RPC

- **DOPU** used to connect to machines exploited by ETERNALCHAMPIONS

# Eternalblue #

[http://bobao.360.cn/learning/detail/3738.html](http://bobao.360.cn/learning/detail/3738.html)

# Eternalromance #

[http://bobao.360.cn/learning/detail/3747.html](http://bobao.360.cn/learning/detail/3747.html)

# fb.py #

fb.py最难受的是python2.6的脚本，所以用起来有点儿麻烦

[https://blog.wanghw.cn/archives/48.html](https://blog.wanghw.cn/archives/48.html)


# Dander Spiritz 工具 #

[http://bobao.360.cn/learning/detail/3739.html](http://bobao.360.cn/learning/detail/3739.html)

[http://bobao.360.cn/learning/detail/3743.html](http://bobao.360.cn/learning/detail/3743.html)