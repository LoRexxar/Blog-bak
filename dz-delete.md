---
title: Discuz!X 3.4 任意文件删除漏洞分析
date: 2017-09-30 15:19:01
tags:
- dz
---

节日快乐！！
<!--more-->


# 0x01 简述

Discuz!X社区软件[1]，是一个采用PHP 和MySQL 等其他多种数据库构建的性能优异、功能全面、安全稳定的社区论坛平台。

2017年9月29日，Discuz!修复了一个安全问题[2]用于加强安全性，这个漏洞会导致前台用户可以导致任意删除文件漏洞。

2017年9月29日，知道创宇404 实验室开始应急，经过知道创宇404实验室分析确认，该漏洞于2014年6月被提交到Wooyun漏洞平台，Seebug漏洞平台收录了该漏洞[3]，漏洞编号ssvid-93588。该漏洞通过配置属性值，导致任意文件删除。

经过分析确认，原有的利用方式已经被修复，添加了对属性的formtype判断，但修复方式不完全导致可以绕过，通过模拟文件上传可以进入其他unlink条件，实现任意文件删除漏洞。

# 0x02 复现

登陆DZ前台账户并在当前目录下新建test.txt用于测试

![image.png-76.2kB][1]
![image.png-172.3kB][2]

请求
```
home.php?mod=spacecp&ac=profile&op=base
POST birthprovince=../../../test.txt&profilesubmit=1&formhash=b644603b
其中formhash为用户hash
```

修改成功之后出生地就会变为../../../test.txt

![image.png-97.2kB][3]

构造请求向`home.php?mod=spacecp&ac=profile&op=base`上传文件（普通图片即可）

![image.png-159kB][4]

请求后文件被删除
![image.png-193.4kB][5]

# 0x03 漏洞分析

Discuz!X的码云已经更新修复了该漏洞

[https://gitee.com/ComsenzDiscuz/DiscuzX/commit/7d603a197c2717ef1d7e9ba654cf72aa42d3e574](https://gitee.com/ComsenzDiscuz/DiscuzX/commit/7d603a197c2717ef1d7e9ba654cf72aa42d3e574)

核心问题在`upload/source/include/spacecp/spacecp_profile.php`

![image.png-113.4kB][6]

跟入代码70行
```
if(submitcheck('profilesubmit')) {
```

当提交profilesubmit时进入判断，跟入177行
![image.png-146.6kB][7]

我们发现如果满足配置文件中某个formtype的类型为file，我们就可以进入判断逻辑，这里我们尝试把配置输出出来看看

![image.png-68.6kB][8]

我们发现formtype字段和条件不符，这里代码的逻辑已经走不进去了

我们接着看这次修复的改动，可以发现228行再次引入语句unlink

```
@unlink(getglobal('setting/attachdir').'./profile/'.$space[$key]);
```

回溯进入条件
![image.png-267.1kB][9]

当上传文件并上传成功，即可进入unlink语句

![image.png-293.8kB][10]

然后回溯变量`$space[$key]`,不难发现这就是用户的个人设置。

只要找到一个可以控制的变量即可，这里选择了birthprovince。

在设置页面直接提交就可以绕过字段内容的限制了。

![image.png-60.1kB][11]

成功实现了任意文件删除

# 0x04 说在最后

在更新了代码改动之后，通过跟踪漏洞点逻辑，我们逐渐发现，该漏洞点在2014年被白帽子提交到Wooyun平台上，漏洞编号wooyun-2014-065513。

由于DZ的旧版代码更新流程不完整，已经没办法找到对应的补丁了，回溯到2013年的DZ3版本中，我们发现了旧的漏洞代码

![image.png-85kB][12]

在白帽子提出漏洞，可以通过设置个人设置来控制本来不可控制的变量，并提出了其中一种利用方式。

厂商仅对于白帽子的攻击poc进行了相应的修复，导致几年后漏洞再次爆出，dz才彻底删除了这部分代码...

期间厂商对于安全问题的解决态度值得反思...

在简单的漏洞分析之后发现，任意文件删除可以删除包括data目录下的install.lock，导致整站重装，配合其他漏洞甚至可以getshell。

# 0x05 Reference

- [1] Discuz!官网
    http://www.discuz.net  
- [2] Discuz!更新补丁
    https://gitee.com/ComsenzDiscuz/DiscuzX/commit/7d603a197c2717ef1d7e9ba654cf72aa42d3e574 
- [3] Seebug漏洞平台收录地址
	https://www.seebug.org/vuldb/ssvid-93588



  [1]: http://static.zybuluo.com/LoRexxar/8xr9kq4bmikxuci286hh6s2l/image.png
  [2]: http://static.zybuluo.com/LoRexxar/9owkr0zm4gwqcpigimhgta6w/image.png
  [3]: http://static.zybuluo.com/LoRexxar/392mpt7s3xw043k783cy41yy/image.png
  [4]: http://static.zybuluo.com/LoRexxar/vk4l8kqfhufghl3maniamczw/image.png
  [5]: http://static.zybuluo.com/LoRexxar/agw0gmgx9byh5hzbr49ouu6c/image.png
  [6]: http://static.zybuluo.com/LoRexxar/hyk311wk0c08ug5hlinfvc4z/image.png
  [7]: http://static.zybuluo.com/LoRexxar/9orn2r2lsgbjtrjtz5eibkgd/image.png
  [8]: http://static.zybuluo.com/LoRexxar/ooma7pdsjfwql1wcqxtlmy1k/image.png
  [9]: http://static.zybuluo.com/LoRexxar/lwyqxqyhen2nw6224v20goel/image.png
  [10]: http://static.zybuluo.com/LoRexxar/e15zw9781j2wt6o5up2g8m71/image.png
  [11]: http://static.zybuluo.com/LoRexxar/54u9dam4i0uhsoriiyreileo/image.png
  [12]: http://static.zybuluo.com/LoRexxar/oijfyp6ogwgwtmcbr2jtb2vc/image.png