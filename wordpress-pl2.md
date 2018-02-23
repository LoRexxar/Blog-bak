---
title: WordPress Updraftplus 插件漏洞2则以及一些有趣的故事
date: 2017-12-19 16:48:13
tags:
- wordpress
- 条件竞争
- ssrf
---

几个月前代码审计的时候，翻了翻wordpress的几个插件，因为wordpress的主要功能基本都是后台的，很少有前台能触发的漏洞，也就没看了，后来顺手就把挖到的2个后台漏洞申请了cve，没想到引来两个人讨论...

你觉得，什么样的漏洞才算是漏洞呢？

<!--more-->


# authentiicated upload file and php code execution #

file `/wp-content/plugins/updraftplus/admin.php` line 1843 function `plupload_action`

via the name parameter to set filename, and move file content into this file.

The server will do a basic verification of the file name, you can get a valid backup file name,just like `backup_2017-11-29-1844_test_d6c634e49869-plugins.zip.`

![image.png-72.3kB][1]

after the 39 lines, this file be delete
![image.png-296.3kB][2]

there are Race condition, when we view this pages before delete after write in. we can make php code execution.


## PoC ##

file name just like:
```
backup_2017-11-29-1844_test_d6c634e49869-plugins
```


file content:
```
<?php
$f = fopen('../a.php','wb');
fwrite($f, '<?php phpinfo();?>');
fclose($f);
```

via upload this file, and view this pages before delete, we can write a a.php into `/wp-content/a.php`

（2017.11.29 Supplement Vulnerability Details）
![image.png-197.1kB][3]

![image.png-105.2kB][4]

![image.png-89.2kB][5]

![image.png-172.5kB][6]

# authentiicated ssrf #

file `/wp-content/plugins/updraftplus/admin.php` line 1233 function updraft_ajax_handler 

when `subaction='httpget'`the curl parameter follow into function http_get,

![image.png-26.1kB][7]

![image.png-163.4kB][8]

they will use curl to request url, it can be exploited to conduct server-side request forgery (SSRF) attacks.



## PoC ##

login and view website
```
http://127.0.0.1/wordpress4.8/wp-admin/options-general.php?page=updraftplus&tab=expert
```

![image.png-8.9kB][9]

use fetch(curl)

![image.png-20.9kB][10]


# 聊聊漏洞 #

由于申请cve的关系，漏洞详情用英文写了，大都是通俗易懂的句子，就不花时间翻译了。

有趣的是，当我在github上公开漏洞详情的时候，引来了两个人的关注（聊天时候的感觉更像是插件的开发者）。

[源地址](https://github.com/LoRexxar/CVE_Request/tree/master/wordpress%20plugin%20updraftplus%20vulnerablity)

第一个人直接向我推了一份mr，让我删除漏洞详情，Σ(っ °Д °;)っ，然后就开始撕逼了...

[https://github.com/LoRexxar/CVE_Request/pull/1](https://github.com/LoRexxar/CVE_Request/pull/1)

整个过程，这个人始终认为漏洞不存在，对话非常有意思，那么，漏洞到底是什么？

```
DavidAnderson684 commented 22 days ago •  edited 

@LoRexxar If somebody can log in as a WP administrator, then they do not need to carry out complicated "attacks" against upload race conditions that only they have access to. Instead, they can just do any of 10,000 other things to manipulate your filesystem, e.g.

Install a WP file manager plugin and manipulate the filesystem at will

Install their own malicious plugin or theme directly using the WP plugin uploader

Upload a malicious backup of their own creation and restore it

Download an existing backup, edit it, upload it and restore it

Use the in-built WordPress code or theme editor to edit existing parts of WordPress

For something to be an attack, it has to give a user powers that they did not already have. A procedure that allows them, through a very complicated work-around method, to do something that they could already do through many other mechanisms is not an attack. Admins can already do the same thing in zillions of other ways. UpdraftPlus is a backup/restore plugin! An evil admin can just create a malicious backup, and then restore it.... he doesn't need to do something really convoluted as an alternative.
```

这段的意思差不多是说，当你可以登录WordPress之后，你有一万种方式可以操作网站的文件系统，而且UpdarPlus本身就是一个用来备份恢复的插件，你完全没必要使用条件竞争来修改文件系统，所以这并不算一个漏洞。

```
你说的当然没错，但是你认为漏洞是什么？

1、在我看来，网站管理员不应当拥有服务器权限。
2、并不是因为有更易于应用的漏洞，别的漏洞就不算是漏洞。
3、wordpress官方对于superadmin的权限一直很模糊，但不意味着superadmin就应该有高于网站管理的权限。
```

那么漏洞应该是什么？

```
DavidAnderson684 commented 22 days ago 

@LoRexxar Whether I think that WordPress should be able to write to the filesystem or not, is not a question for me in my position. I am not part of the WordPress core team. If WordPress could not write to the filesystem, then a backup/restore plugin would be theoretically impossible. We can only deal with WP as it is. In WP's security model, WP can write to the filesystem and this is not considered a defect; and admins are all powerful (N.B. a "super admin" is the all-powerful admin on a WP multisite install; but UD's code already handles that correctly too).

Note also that your attack cannot work for a further reason. The code cannot write directly into wp-content... the description just assumes you can write there and perform a directory traversal . But the file location of the file during upload is PHP's temporary directory... and after that, it gets moved to a file ending in '.zip.tmp' in a directory that's protected with an .htaccess file, with directory traversal prevented by use of basename(). There's no mechanism for achieving that, or for running a .zip.tmp file as a PHP file. It seems that you've not tried to actually carry out this 'attack' in practice?

For me, the important thing is just that you retract the report and the CVE application. Once a CVE number gets granted, it then appears in lots of automated security tools. We then have a problem of 1 million active users who are going to be scared, and it will create a huge avalanche of support requests and negative publicity for us. So please can you retract it as soon as possible? Thank you!
```

然后他提到了一个问题，WordPress的官方始终认为，超级管理员应该管理好自己的网站，对自己的网站负责，所以WordPress的官方本身就给了可以操作文件系统的权限，所以可以修改文件系统，并不应该算作漏洞。

中间一段说的是有关漏洞利用的一些问题，稍后我们会再次提到。

最后一段就比较有意思了，他认为在我们讨论出一个结果之前，我不应该草率的申请cve id，那会导致很多用户害怕。（这时候我才开始明白为什么他总是在推脱漏洞不存在，应该是插件的开发者？）

```
1、首先，我已经取得了CVE i，CVE-2017-16870、CVE-2017-16871，漏洞存在的理由我已经描述的相当清楚。
2、其次，作为安全研究者，我的目的是研究系统是否存在漏洞，并不是所有漏洞都一定易于应用！
3、最后，既然你认为我所说的没有道理，插件没有漏洞，那你完全不必这么紧张。

我认为作为开发者，正确的面对安全问题才是好的态度（如果你不认为这是安全问题，那你可以什么都不做！），竞争上传漏洞，完全可以通过先验证，再上传的方式修复，SSRF漏洞可以通过限制内网请求来避免安全问题！攻击者可以通过SSRF漏洞攻击内网的各种设备，我想我不需要给你科普SSRF是什么！
```

不得不说，开发和安全工作者对于漏洞的看法天差地别，他始终认为漏洞不是插件的问题，但却一直试图让我不公开漏洞。

就像那句话说的，如果事情真的不是那样的，那又何必有鬼呢？

后面的对话就是在扯一些别的问题了，出现了一个新的人，他提出插件上传文件使用了`wp_handle_upload()`，这个函数压根不可以上传php文件。

可笑的是，在浪费了2个小时重新温习漏洞之后，我再次复现了漏洞，而且是最新版本：>

![image.png-121.5kB][11]

漏洞的触发逻辑是，文件在上传之后，经过了一次改名，然后才经过了判断...事实上，上传文件名完全可以是合法的。

再后来的讨论已经失去意义了，所以我关闭了pr，但是，你觉得什么样的漏洞才算是漏洞呢？

下面是他最后的回复
```
DavidAnderson684 commented 20 days ago 

So, what's your problem with the authenticity of the vulnerability?
You've asked why this is not a vulnerability . I've already answered that; it's because a vulnerability is something that you did not already have access to do through other mechanisms, by design. A vulnerability cannot include something in which the highest-level user deliberately tries to hurt himself. Beyond these scopes, we are only talking about inconsequential opinions about coding style.

The following is not a security hole in WordPress:

Log in as an admin
Edit all site content in the posts editor and replace it with the words "I HACKED YOUR SITE!!!"
Do you understand why that it is not a security hole in WordPress's posts editor?

Or to give another example. WordPress core, by deliberate design, does not allow people at the "author" level to inject JavaScript into a post. It does, however, allow them to inject it if they are an admin. That is because admins are all-powerful. They can already break the site by any method that they like.

So, at best, you are describing "a way for an admin to achieve something that he is already intentionally allowed to achieve 10,000 other ways (eg. restoring an evil backup)". That is not what security researchers label as a vulnerability.
```

在和这个人讨论了2天之后，我重新审视了自己对漏洞的看法，漏洞的定义在维基百科上，是这样的。

```
计算机安全隐患（英语：Vulnerability），俗称安全漏洞（英语：Security hole），指计算机系统安全方面的缺陷，使得系统或其应用数据的保密性、完整性、可用性、访问控制和监测机制等面临威胁。

许多安全漏洞是程序错误导致的，此时可叫做程序安全错误（Security bug），但并不是所有的安全隐患都是程序安全错误导致的。
```

对于第一个漏洞来说，上传备份文件是那个功能的正常功能，限制上传文件的类型是应有的安全限制，竞争导致这个安全限制被绕过，漏洞发生！

对于第二个漏洞来说，httpwget应是用来测试网络是否通的工具，但没有对内网请求做任何限制，致使SSRF漏洞可以攻击内网，漏洞存在！

那么，你觉得漏洞是不是存在呢？



  [1]: http://static.zybuluo.com/LoRexxar/ypg8xnr3sond8644y8swl8jc/image.png
  [2]: http://static.zybuluo.com/LoRexxar/68a8puqj0k96ifgmryzi2ogw/image.png
  [3]: http://static.zybuluo.com/LoRexxar/ewecs9lrot08ijg2vlvituh9/image.png
  [4]: http://static.zybuluo.com/LoRexxar/u7y5hg3m1y2nj2c8kxbbhuk6/image.png
  [5]: http://static.zybuluo.com/LoRexxar/8o61e5tla9dglj0m5g2z7349/image.png
  [6]: http://static.zybuluo.com/LoRexxar/r1pfhrohhin6kj295zoqckjh/image.png
  [7]: http://static.zybuluo.com/LoRexxar/nvvhoe3gxmepnug6yf6aaojr/image.png
  [8]: http://static.zybuluo.com/LoRexxar/pksfa3e4gtjg6ztgnuubedkb/image.png
  [9]: http://static.zybuluo.com/LoRexxar/smc72erghubs8a1xvtzo5sjp/image.png
  [10]: http://static.zybuluo.com/LoRexxar/qa12ejx4ad0pf4c245fe3nqg/image.png
  [11]: http://static.zybuluo.com/LoRexxar/gm5rvjq13ajcw3o19cyprllv/image.png
