title: Nsctf2015_Writeup
date: 2015-09-28 00:16:10
tags:
- Blogs
- ctf
categories:
- Blogs
---
从9月26号开始接连撸了3个ctf，nsctf，gctf，趋势ctf，虽然实力还有待提高，但是幸运的是nsctf意外撸进了前50名，也做了不少web题，这里先写下nsctf的writeup留存.

<!--more-->
这边贴上最终的名次和完成情况，算是留个纪念。
![](/img/nsctf_writeup/1.png)
![](/img/nsctf_writeup/2.png)

# MISC #

## 0x01 Twitter Point: 100 (Done) ##
签到题，没什么好说的。
关注NSCTF twitter，然后得到hash
fc42aa2046ed6e90cab82b1094b19adb
MD5解密得到nsfocus666
按格式加入，Get！

## 0x02 WireShark Point: 250 ##
题目是分析数据包，实在不会，用wireshark翻了翻得到有个key.rar，多了也不会了，这里贴上ddog的代码（不知道我缺了什么步骤，反正是没用...)
```
import rarfile
import os

from threading import Thread
import time
a = range(10000,100000)

#def genaratePass():
#    for i in a:
#        return "nsfocus" + str(i)

def pojie_rar():
    path = './key.rar'
    rar = rarfile.RarFile("key.rar", mode='r')
    for i in a:
        password = 'nsfocus' + str(i)
        #print password
        try:
            rar.extractall(path="./",  pwd=password)
            print ' ----success!,The password is %s' % password
            rar.close()
            return True
        except:
            pass
        #print ''

#def genaratePass():
#for i in xrange(00000, 99999):
       # return 'nsfocus' + str(i)

if __name__ == '__main__':
    pojie_rar()
```


## 0x03 小绿的女神 Point: 300 (Done) ##
题目开始没做，原题没有截图，大概是说女神的生日是2月8号，想为女神充值到208元，查询余额得到208就可以得到flag，说实话十六进制仅仅是能看懂而已，后来z神告诉我原理，这里贴上，也算是多了一个经历。

这题主要的问题就是，卡中有3点很关键：
1、消费的金额
2、这张卡的金额上限
3、这张卡的余额
**2=1+3**
卡的结构大概是这样的
![](/img/nsctf_writeup/3.png)

根据描述应该很容易看出结构，是S ~S S ~S这样的结构
分析得到第一部分是消费的金额，第二部分是卡的总额也就是上限，第三部分是卡的余额。
很重要的一点就是**16进制数字在内存中是小端对齐存放的**
所以![](/img/nsctf_writeup/5.jpg)比如这个其实是0xFFFFD5D0

所以最后修改的样子就是这样的：
![](/img/nsctf_writeup/4.png)
提交 Get Flag!


# crypto #

## 0x01 神奇的字符串 Point 100 (Done) ##
题目很简单，只不过一开始看到懵了，后来把头拖去百度，得到AES加密，得到flag，然后后面按照NSCTF的格式位移，Get Flag!

## 0x02 神奇的图片 Point 100 ##
这道题是一张图片题，开始用Stegsolve翻了翻没什么收获，想到可能是两张图片合起来，但是我找不到图片的头，所以最后只能放弃这题

## 0x03 神秘的图片+10086 Point 200 (Done) ##
图片隐写，用Stegsolve翻翻找到一张很像二维码的图片，和正常的比对发现黑白换过了，用ps处理扫码Get Flag


# web #

## 0x01 Be careful Point 100 (Done) ##
这题目简直坑，打开页面什么东西都没有，翻什么都得不到有用的信息，最后脑洞加Index.php和index.html，发现Index.php会发生跳转，到index.html，于是拦截查看源码，Get flag!

## 0x02 Where are you come from Point 100 (Done) ##
题目简直狗血，打开弹出来一个你来自火星吗？第一反应是改refferer，怎么改都没用，后面除了hit:只有本机才能访问，于是在header中添加了x-forwarded-for，开始一直以为是127.0.01，无意间脑洞nsctf官网ip，Get flag!
ps:主办方简直有病。。。

## 0x03 Version Point 100 (Done) ##
打开题目，是一个提交php版本的题目，第一反应是burp跑php版本，各种不出。官方hit：提示就是普通的php版本，不用多想。无意间得到提示...
http://www.nsctf.net:8000/fa81bb665474f11c025b5355582af315/web/03/?ver=5.5.9-1ubuntu4.12

Get Flag...不多说了，这脑洞简直神人才能想到

## 0x04 Brute force Point:200 (Done) ##
看名字就知道是要burp跑密码，看到title是password.txt，打开发现密码表，burp跑出得到密码，登录得到了域名：
http://www.nsctf.net:8000/fa81bb665474f11c025b5355582af315/web/04/290bca70c7dae93db6644fa00b9d83b9.php?act=add

进入是要使用管理员留言就可以得到flag，发现了隐藏参数是用户名，输入了admin留言不行，各种注入各种xss不行，结果无意间试了root...
GetFlag！

## 0x05 Decode Point:200 (Done) ##
题目没什么好说的，百度下各个函数就知道是什么意思了，构造脚本Get flag。
```
<?php

$x=base64_decode(fjg4OjM2ZTFiZzg0MzhlNDE3NTdkOjI5Y2dlYjZlNDhjYEdVRFRPfDtoYm1n);

for($i=0;$i<strlen($x);$i++){
	$c = substr($x,$i,1);
	$y = ord($c)-1;
	$c = chr($y);
	$key = $key.$c;
}
$key = strrev($key);
echo $key;
?>
```

## 0x06 javascript Point:200 (Done) ##
看到javascript估计已经能想到是什么题目了，js代码去混淆。这时候用到一个神器，就是firefox控制台（当然chrome也可以）
把这一堆乱七八糟的东西拖去控制台，然后美化源码，得到能看的函数声明：
```
eval(function (p, a, c, k, e, d) {
  e = function (c) {
    return (c < a ? '' : e(parseInt(c / a))) + ((c = c % a) > 35 ? String.fromCharCode(c + 29)  : c.toString(36))
  };
  if (!''.replace(/^/, String)) {
    while (c--) d[e(c)] = k[c] || e(c);
    k = [
      function (e) {
        return d[e]
      }
    ];
    e = function () {
      return '\\w+'
    };
    c = 1;
  };
  while (c--) if (k[c]) p = p.replace(new RegExp('\\b' + e(c) + '\\b', 'g'), k[c]);
  return p;
}
```
至于跟在后面的东西，可以知道就是对应的参数p,a,c,k,e,d，别被这乱七八糟的东西吓到，找到几个有用的，
就是一个p，在单引号中间的东西，拖到控制台，美化源代码，得到下面的东西。
```
_f = function () {
  var f = document.createElement('form');
  document.getElementById('login').appendChild(f);
  f.name = 'login';
  return f
}();
_uname = function () {
  var uname = document.createElement('input');
  uname.type = 'text';
  uname.id = 'uname';
  uname.value = 'Input Username';
  uname.style.margin = '0px 0px 0px 60px';
  _f.appendChild(uname);
  uname.onfocus = function () {
    if (this.value == 'Input Username') this.value = ''
  };
  uname.onblur = function () {
    if (this.value == '') this.value = 'Input Username'
  };
  return uname
}();
_br = function () {
  var br = document.createElement('br');
  _f.appendChild(br);
  br = document.createElement('br');
  _f.appendChild(br);
  return br
}();
_upass = function () {
  var upass = document.createElement('input');
  upass.type = 'password';
  upass.id = 'upass';
  upass.value = 'Input Password';
  upass.style.margin = '0px 0px 0px 60px';
  _f.appendChild(upass);
  upass.onfocus = function () {
    if (this.value == 'Input Password') this.value = ''
  };
  upass.onblur = function () {
    if (this.value == '') this.value = 'Input Password'
  };
  return upass
}();
_btn = function () {
  var btn = document.createElement('input');
  _f.appendChild(btn);
  btn.type = 'button';
  btn.value = 'login';
  btn.onclick = function () {
    uname = document.getElementById('uname').value;
    upass = document.getElementById('upass').value;
    if (uname == '') alert('Please Input Username!');
     else if (upass == '') alert('Please Input Password!');
     else {
      eval(unescape('var%20strKey1%20%3D%20%22JaVa3C41ptIsAGo0DStAff%22%3B%0Avar%20strKey2%20%3D%20%22CaNUknOWThIsK3y%22%3B%0Avar%20strKey3%20%3D%20String.fromCharCode%2871%2C%2048%2C%20111%2C%20100%2C%2033%29%3B%0Aif%20%28uname%20%3D%3D%20%28strKey3%20+%20%28%28%28strKey1.toLowerCase%28%29%29.substring%280%2C%20strKey1.indexOf%28%220%22%29%29%20+%20strKey2.substring%282%2C%206%29%29.toUpperCase%28%29%29.substring%280%2C%2015%29%29%29%20%7B%0A%20%20%20%20var%20strKey4%20%3D%20%27Java_Scr1pt_Pa4sW0rd_K3y_H3re%27%3B%0A%20%20%20%20if%20%28upass%20%3D%3D%20%28strKey4.substring%28strKey4.indexOf%28%271%27%2C%205%29%2C%20strKey4.length%20-%20strKey4.indexOf%28%27_%27%29%20+%205%29%29%29%20%7B%0A%20%20%20%20%20%20%20%20alert%28%27Login%20Success%21%27%29%3B%0A%20%20%20%20%20%20%20%20document.getElementById%28%27key%27%29.innerHTML%20%3D%20unescape%28%22%253Cfont%2520color%253D%2522%2523000%2522%253Ea2V5X0NoM2NrXy50eHQ%3D%253C/font%253E%22%29%3B%0A%20%20%20%20%7D%20else%20%7B%0A%20%20%20%20%20%20%20%20alert%28%27Password%20Error%21%27%29%3B%0A%20%20%20%20%7D%0A%7D%20else%20%7B%0A%20%20%20%20alert%28%27Login%20Failed%21%27%29%3B%0A%7D'))
    }
  };
  return false
}();
```
相信很容易就能看出问题了，然后把eval整个复制进控制台，把eval改为alert或者直接document.write,得到关键源码：
```
var strKey1 = "JaVa3C41ptIsAGo0DStAff";
var strKey2 = "CaNUknOWThIsK3y";
var strKey3 = String.fromCharCode(71, 48, 111, 100, 33);
if (uname == (strKey3 + (((strKey1.toLowerCase()).substring(0, strKey1.indexOf("0")) + strKey2.substring(2, 6)).toUpperCase()).substring(0, 15))) {
    var strKey4 = 'Java_Scr1pt_Pa4sW0rd_K3y_H3re';
    if (upass == (strKey4.substring(strKey4.indexOf('1', 5), strKey4.length - strKey4.indexOf('_') + 5))) {
        alert('Login Success!');
        document.getElementById('key').innerHTML = unescape("%3Cfont%20color%3D%22%23000%22%3Ea2V5X0NoM2NrXy50eHQ=%3C/font%3E");
    } else {
        alert('Password Error!');
    }
} else {
    alert('Login Failed!');
}
```
这时候如果你直接把unscape中的东西拿出来访问的话，他会提示你的用户名错误，本来以为是cookie的问题，所以回去跑出
uname=G0od!JAVA3C41PTISAGO
upass=1pt_Pa4sW0rd_K3y_H3re
登录仍然无果，脑洞一开post数据，Get Flag!

## 0x07 social engeer Point:150(Done) ##

题目很扯淡，给出小明的名字和生日还有qq，需要跑出密码，各种尝试无果官方给出了hit。
hit:很简单的中文密码，姓名加生日，注意大小写。
于是写字典跑，得到了密码Xiaoming09231995，这时候出现了第二步（最坑的地方），
题目要求通过电话社工王先生的信息，这里只是提到了社工，但是并没有说到什么程度会得到flag，于是得到了一大堆信息：
1、微信加好友找一下，伟
2、支付宝转账搜一下账号，王伟 浙江嘉兴
3、手机qq搜一下 王欣薇爸爸 31岁 浙江杭州
4、身份证：34112519831224875X 
5、出生年月日：19831224
6、身份证头得到的是  安徽省滁州市定远县

最后被逼的没办法还发了短信，结果发现这是真人（官方和王伟到底什么愁什么怨），最后开脑洞，想到可能还是check的地方有问题，输入身份证Get Flag!

ps:别拦着我，我要报警了

## 0x08 LFI Point:200(Done) ##

没什么可说的，文件上传
php://filter/read=convert.base64-encode/resource=index.php
```
<?php
error_reporting(0);

if(isset($_POST['submit'])){

  if(isset($_POST['file'])){

          $file = $_POST['file'];

          $method = explode("=", $file);

          if( ($method[0] == "php://filter/read") && ($method[2] == "index.php") ){
           include($file);
        exit();
    }else{
            exit('error file or error method');
    }
  }
}
```

## 0x09 change password Point:300 (Done) ##

尝试了下，感觉应该是有源码，于是找到.index.php.swp源码，这里源码忘记保存下来了，审计代码构造payload,这里有个坑，也不知道主办方是不是脑子有坑。。。id要等于1，惯性思维是cookie里面发现的原密码150923和id=3

**payload:userInfo=a:2:{s:2:"id";i:1;s:4:"pass";s:8:"20150923";}&oldPass=20150923&newPass=321321**

Get Flag!

## 0x0A Variable cover Point:250 (Done) ##
题目开始完全没有思路，注入完全过不去，官方给出hit:不按常理的备份文件。
然后各种开脑洞，无意间发现Index.php.，源码得到开始构造payload。

**payload:username='qwe&password=||1#&Submit=%E6%8F%90%E4%BA%A4&_CONFIG=123321**

这里username=' => username=\' => username[0]=\

所以Get Flag!

## 0x0B File Upload Point:400 (Done) ##

## 0x0C SQLI Point:350 (Done) ##
稍微翻了翻源码，发现了隐藏参数filtername，测试下发现：
filtername是过滤器，可以通过设置filtername绕过对username的部分过滤
username=teadminst&filtername=admin    会先过滤再查询

剩下就是对单引号的过滤，这里用了一个黑科技%2527,这里%25被解析成%，然后单引号成功，于是构造payload；
```
username=admin%2527+uniewqoN+Aewqll+sEleewqct+(group_concat(flag)),2+from+flag#&filtername=ewq
```

Get Flag!




总体说来这次nsctf收获很多，知道很多以前不知道的黑科技，经过一个假期的学期，对原来很多不懂得东西有了新的认识，还有z神和呵呵抬我一手，进了前50，很开心（0。0）

