---
title: DeDeCMS v5.7 密码修改漏洞分析
date: 2018-01-19 11:18:51
tags:
- dedecms
- 逻辑漏洞
---

文章首发在paper.seebug.org平台


# 0x01 背景

织梦内容管理系统(DedeCms)以简单、实用、开源而闻名，是国内最知名的PHP开源网站管理系统，也是使用用户最多的PHP类CMS系统，在经历多年的发展，目前的版本无论在功能，还是在易用性方面，都有了长足的发展和进步，DedeCms免费版的主要目标用户锁定在个人站长，功能更专注于个人网站或中小型门户的构建，当然也不乏有企业用户和学校等在使用该系统。

2018年1月10日， 锦行信息安全公众号公开了一个关于DeDeCMS前台任意用户密码修改漏洞的细节[2]。

2018年1月10日，Seebug漏洞平台[3]收录该漏洞，漏洞编号为SSV-97074，知道创宇404漏洞应急团队成功复现该漏洞。

2018年1月17日，阿里先知平台公开了一个任意用户登陆漏洞[4][1]，和一个安全隐患[6]，通过组合漏洞，导致后台密码可以被修改。

2018年1月18日，知道创宇404漏洞应急团队成功复现该漏洞。



# 0x02 漏洞简述

整个漏洞利用链包括3个过程：
1、前台任意用户密码修改漏洞
2、前台任意用户登陆漏洞
3、前台管理员密码修改可影响后台的安全隐患

通过3个问题连起来，我们可以重置后台admin密码，如果我们获得了后台地址，就可以进一步登陆后台进行下一步攻击。

## 1、前台任意用户密码修改漏洞
前台任意用户密码修改漏洞的核心问题是由于DeDeCMS对于部分判断使用错误的弱类型判断，再加上在设置初始值时使用了NULL作为默认填充，导致可以使用弱类型判断的漏洞来绕过判断。

漏洞利用有几个限制：
1、漏洞只影响前台账户
admin账户在前台是敏感词无法登陆
2、admin账户的前后台密码不一致，无法修改后台密码。
3、漏洞只影响未设置密保问题的账户

## 2、前台任意用户登陆漏洞

前台任意用户登陆漏洞主要是利用了DeDeCMS的机制问题，通过一个特殊的机制，我们可以获得任意通过后台加密过的cookie，通过这个cookie我们可以绕过登陆，实现任意用户登陆。

漏洞利用有一个限制：
如果后台开启了账户注册审核，那就必须等待审核通过才能进行下一步利用。

## 3、前台管理员密码修改可影响后台的安全隐患

在DeDeCMS的设计中，admin被设置为不可从前台登陆，但是当后台登陆admin账户的时候，前台同样会登陆管理员账户。

而且在前台的修改密码接口，如果提供了旧密码，admin同样可以修改密码，并且这里修改密码会同步给后台账户。

通过3个漏洞配合，我们可以避开整个漏洞利用下的大部分问题。

前台任意用户密码修改漏洞->修改admin密码，前台任意用户登录漏洞->登陆admin账户，通过刚才修改的admin密码，来重置admin账户密码。

# 0x03 漏洞复现

## 1、 登陆admin前台账户

安装DeDeCMS
![image.png-204.3kB][2]

注册用户名为000001的账户

![image.png-87.2kB][3]

由于是本地复现漏洞，所以我们直接从数据库中修改为审核通过
![image.png-58.8kB][4]

访问
```
http://your_website/member/index.php?uid=0000001
```

![image.png-375.6kB][5]

获取cookie中`last_vid_ckMd5`值，设置

```
DeDeUserID_ckMd5为刚才获取的值，并设置DedeUserID为0000001
访问
http://your_website/member/
```
![image.png-295.3kB][6]

## 2、修改admin前台登陆密码

使用DeDeCMS前台任意用户密码修改漏洞修改admin前台密码。
构造漏洞利用请求

```
http://yourwebsite/member/resetpassword.php

dopost=safequestion&safequestion=0.0&safeanswer=&id=1
```
![image.png-62kB][7]

从Burp获取下一步利用链接
```
/member/resetpassword.php?dopost=getpasswd&id=1&key=nlszc9Kn
```
![image.png-242.2kB][8]

直接访问该链接，修改新密码
![image.png-179.8kB][9]

成功修改登陆admin密码

## 3、修改后台密码

访问

```
http://yourwebsite/member/edit_baseinfo.php
```

使用刚才修改的密码再次修改密码
![image.png-147.4kB][10]
 
 成功登陆
![image.png-312.4kB][11]
 

# 0x04 代码分析

## 1、 前台任意用户登陆

在分析漏洞之前，我们先来看看通过cookie获取登陆状态的代码。

```
/include/memberlogin.class.php 161行
```
![image.png-357.4kB][12]

通过GetCookie函数从`DedeUserID`取到了明文的M_ID，通过`intval`转化之后，直接从数据库中读取该id对应的用户数据。

让我们来看看`GetCookie`函数

```
/include/helpers/cookie.helper.php 56行
```
![image.png-183.8kB][13]

这里的`cfg_cookie_encode`是未知的，DeDeCMS通过这种加盐的方式，来保证cookie只能是服务端设置的，所以我们没办法通过自己设置cookie来登陆其他账户。

这里我们需要从别的地方获取这个加密后的值。

```
/member/index.php  161行
```

![image.png-305.1kB][14]

161行存在一段特殊的代码，这段代码是用来更新最新的访客记录的，当`last_vid`没有设置的时候，会把`userid`更新到这个变量中，更新到flag中。

而这里的`userid`就是注册时的用户名（如果不是已存在的用户名的话，会因为用户不存在无法访问这个页面）。

通过这种方式，我们就可以通过已知明文来获取我们想要的密文。

这里我们通过注册`userid`为形似00001或者1aaa这样的用户，在获取登陆状态时，`mid`会经过`intval`的转化变为1，我们就成功的登陆到admin的账户下。

ps：该漏洞影响所有用户

## 2、前台任意用户密码修改

漏洞主要逻辑在

/member/resetpassword.php 75行至95行

![image.png-198.5kB][15]

当找回密码的方式为安全问题时

dedecms会从数据库中获取用户的安全问题、回答进行比对，当我们在注册时没设置安全问题时。

从数据库中可以看到默认值为NULL（admin默认没有设置安全问题）

![image.png-7.2kB][16]

下面是设置了安全问题时数据库的样子，safequestion代表问题的id，safeanswer代表安全问题的回答。

我们需要绕过第一个判断
```
if(empty($safequestion)) $safequestion = '';
```

这里我们只要传入`0.0`就可以绕过这里，然后`0.0 == 0`为True，第二个判断`NULL==""`为True，成功进入sn函数。

跟入`/member/inc/inc_pwd_functions.php` 第150行

![image.png-205.6kB][17]

有效时间10分钟，进入newmail函数

跟入`/member/inc/inc_pwd_functions.php` 第73行

![image.png-465.2kB][18]

77行通过random生成了8位的临时密码。

这里我们使用的是安全问题修改密码，所以直接进入94行，将key代入修改页。

跳转进入形似
```
/member/resetpassword.php?dopost=getpasswd&amp;id=1&amp;key=nlszc9Kn
```
的链接，进入修改密码流程

唯一存在问题的是，这里`&`错误的经过一次编码，所以这里我们只能手动从流量中抓到这个链接，访问修改密码。

## 3、修改后台密码安全隐患

在DeDeCMS的代码中，专门对前台修改管理员密码做了设置，如果是管理员，则一并更新后台密码，也就是这个安全隐患导致了这个问题。

```
/member/edit_baseinfo.php 119行
```

![image.png-165.3kB][19]

# 0x05 修复方案

截至该文章完成时，DeDeCMS的官方仍然没有修复该漏洞，所以需要采用临时修复方案，等待官方正式修复更新。

由于攻击漏洞涉及到3个漏洞，但官方仍然没有公开补丁，所以只能从一定程度上减小各个漏洞的影响。
  
- 前台任意用户登陆漏洞：开启新用户注册审核，当发现userid为1xxxx或1时，不予以
通过审核。

在官方更新正式补丁之前，可以尝试暂时注释该部分代码，以避免更大的安全隐患
```
/member/index.php 161-162行
```

![image.png-141.9kB][20]


- 前台修改后台管理员密码：设置较为复杂的后台地址，如果后台地址不可发现，则无法登陆后台。
- 前台任意用户密码修改漏洞：

修改文件`/member/resetpassword.php` 第84行
![image.png-133.6kB][21]

将其中的==修改为===

![image.png-117.4kB][22]

即可临时防护该该漏洞。

# 0x06 ref

- [1] DeDeCMS官网
http://www.dedecms.com/
- [2] 漏洞详情原文 
https://mp.weixin.qq.com/s/2ULQj2risPKzskX32WRMeg
- [3] Seebug漏洞平台
	https://www.seebug.org/vuldb/ssvid-97074
- [4] 阿里先知平台漏洞分析1
	https://xianzhi.aliyun.com/forum/topic/1959
- [5] 阿里先知平台漏洞分析2
	https://xianzhi.aliyun.com/forum/topic/1961
- [6] 漏洞最早分析原文
http://www.cnblogs.com/iamstudy/articles/dedecms_old_version_method.html


  [1]: http://static.zybuluo.com/LoRexxar/fhs5a1xzcs9wlmo79dz1fa7t/image.png
  [2]: http://static.zybuluo.com/LoRexxar/35x2i03cf1o6dx5zinorhxkg/image.png
  [3]: http://static.zybuluo.com/LoRexxar/i5cw04ts7pwt2knzqdvwsxa9/image.png
  [4]: http://static.zybuluo.com/LoRexxar/cvx4w4vakv5zawbqbn5blw9a/image.png
  [5]: http://static.zybuluo.com/LoRexxar/baigsn6cfc45rqqdp2ij002z/image.png
  [6]: http://static.zybuluo.com/LoRexxar/m6cdhwm8x5fru0ghzteugskb/image.png
  [7]: http://static.zybuluo.com/LoRexxar/r3vnslnl4xvhgceu63fx2ams/image.png
  [8]: http://static.zybuluo.com/LoRexxar/ed4f7dpso9ibg5ppwgct82m5/image.png
  [9]: http://static.zybuluo.com/LoRexxar/3hc7ji53z9l39rckak3ezqsu/image.png
  [10]: http://static.zybuluo.com/LoRexxar/y61j1o170sf6nwmwkn4gop45/image.png
  [11]: http://static.zybuluo.com/LoRexxar/z0vs7wayrvpcwctxbtc0szau/image.png
  [12]: http://static.zybuluo.com/LoRexxar/r4505g75o8jkyp8gn9iutgx4/image.png
  [13]: http://static.zybuluo.com/LoRexxar/c0nkya1tqtl2huoc4hhj0el9/image.png
  [14]: http://static.zybuluo.com/LoRexxar/fc0z0lfq4jdnk0td1abbhdyk/image.png
  [15]: http://static.zybuluo.com/LoRexxar/mmf74nzg4idqncfcku6iqny3/image.png
  [16]: http://static.zybuluo.com/LoRexxar/vnj84gtz8bdknys5m7fa290x/image.png
  [17]: http://static.zybuluo.com/LoRexxar/zvoi03jzfdaaml0dfoqhnlq0/image.png
  [18]: http://static.zybuluo.com/LoRexxar/465zcukntsl4x56x6y0fzlzx/image.png
  [19]: http://static.zybuluo.com/LoRexxar/e9c2lkab7t9j2g9lblrsk9be/image.png
  [20]: http://static.zybuluo.com/LoRexxar/7egs2kovzmftad0551ny60ez/image.png
  [21]: http://static.zybuluo.com/LoRexxar/2mql17fthqbkiu6z8scjwgai/image.png
  [22]: http://static.zybuluo.com/LoRexxar/2dj52izgb34tpgvhu6s9p8cv/image.png
