---
title: 浅谈xss的后台守护问题
date: 2016-12-24 21:55:48
tags:
- Blogs
- xss
categories:
- Blogs
---

在出好HCTF2016的两道xss题目后，就有了一个比较严重的问题就是，如何守护xss的后台，用不能人工一直在后台刷新吧（逃

<!--more-->

一般来说，之所以python的普通爬虫不能爬取大多数的网站的原因，是因为大多数网站都把显示数据的方式改成了js执行，通过各种各样的方式，然后输出到页面中，浏览器一般帮助你完成这部分js的解析，所以我们使用的时候，就感受不到阻碍了。

但是对于普通的爬虫来说，这就是比较致命的了，那么对于python的爬虫来说，我们一般使用比较轻量级的**selenium+phantomjs**来解决，但是如果你的xss题目对浏览器内核有需求呢？

就好像我这里的题目guestbook浏览器要求必须是chrome一样，所以我这里选择了**selenium+webdriver**来解决。

首先第一个问题就是你的电脑里必须要有对应的浏览器，如果想只用chrome的webdriver就必须安装过chrome，如果想用firefox的同理。

幸运的是，有份官方文档给我们看

[http://www.seleniumhq.org/docs/03_webdriver.jsp](http://www.seleniumhq.org/docs/03_webdriver.jsp)

有个比较重要的就是firefox的webdriver是自带的，但是chrome并不是，所以我们需要自己来下载一个

[https://sites.google.com/a/chromium.org/chromedriver/downloads](https://sites.google.com/a/chromium.org/chromedriver/downloads)

ps: webdriver的版本和本机chrome相符合的，而且语法也有所变化，这里推荐最新版chrome+最新版webdriver

pps: 虽然我没找到哪里有明确的描述，但是事实上，启动webdriver的时候，webdriver会像浏览器一样弹出来，在我的测试下，在没桌面的情况下怎么都运行不起来，可能是需求桌面的，所以想要放在线上服务器的话，可能需要有桌面才可以（我想没人会在线上服务器装个桌面吧，这里估计还是windows服务器）

## 一个普通的守护脚本 ##

ok，到了最头疼的问题了，如何处理选手插入的js，如果你尝试了用上面的办法写一个守护脚本，你会发现，选手发一个`alert(1)`，你的代码就会卡住，然后bot就挂了，这里我使用了通过不停的点击确定，直至捕获错误为止

```

	#!/usr/bin/env python
	# -*- coding:utf-8 -*-
	
	import selenium
	from selenium import webdriver  
	from selenium.webdriver.common.keys import Keys  
	from selenium.common.exceptions import WebDriverException
	import os 
	import time 
	
		
	while 1:
		chromedriver = "C:\Users\Administrator\AppData\Local\Google\Chrome\Application\chromedriver.exe"  
		os.environ["webdriver.chrome.driver"] = chromedriver  
		browser = webdriver.Chrome(chromedriver)  
	
		url = "http://guestbook.hctf.io/admin_lorexxar.php"  	
		browser.get(url)
	
	
		browser.add_cookie({'name': 'admin',
		 'value' : 'hctf2o16com30nag0gog0',
		 'path' : '/'})
		browser.get(url)
	
		while 1:
			try:
				browser.switch_to_alert().accept()
	
			except selenium.common.exceptions.NoAlertPresentException:
				break
	
		print browser.title
		print time.strftime("%Y-%m-%d %X", time.localtime())
		time.sleep(2)
		browser.quit()
		time.sleep(10)
```

这里的
```

	browser.switch_to_alert().accept()
```
可以处理一切的弹窗问题，保证webdriver起码不会被弹窗卡住

```

	print browser.title
	print time.strftime("%Y-%m-%d %X", time.localtime())
	time.sleep(2)
```
这里输出browser.title的原因是，这里如果不调用browser输出页面内容的话，如果因为网络原因，页面还没有加载出来，这里会经过下面的`time.sleep`直接退出。

等待页面加载完成后，我们需要给时间来加载选手的js，所以这里的`time.sleep`是必须的。

在我的测试下，这里只要没有弹窗，即使js没有加载完成，也会被quit关闭webdriver。

由于留给加载js的时间是有限的，所以在这里，需要另一个脚本来清空数据库中发送的留言，这里我把这部分单独出去了，不过完全可以集合在脚本里，就不多提了。

## 需要登陆或者需要交互式的xss守护脚本 ##

上面说了，类似于留言板的守护方式，那么如果是交互式的，而且通过session来判断用户的，该怎么办呢？

这里我使用request来登陆获取cookie，然后传给browser中

```

	#!/usr/bin/env python
	# -*- coding:utf-8 -*-
	
	import selenium
	from selenium import webdriver  
	from selenium.webdriver.common.keys import Keys  
	from selenium.common.exceptions import WebDriverException
	import os 
	import time 
	import requests
	
	while 1:
		s = requests.Session()
		url = 'http://sguestbook.hctf.io/login.php'
		data = {
			'user': 'admin',
			'pass': 'jklfdnkrejknklhjklfjql'
		}
	
		r = s.post(url, data, allow_redirects = False)
	
		session = r.headers['Set-Cookie'][10:-8]
	
		chromedriver = "C:\Users\Administrator\AppData\Local\Google\Chrome\Application\chromedriver.exe"  
		os.environ["webdriver.chrome.driver"] = chromedriver  
		browser = webdriver.Chrome(chromedriver)  
	
		url = "http://sguestbook.hctf.io/user.php"  	
		browser.get(url)
	
		browser.switch_to_alert().accept()
		browser.add_cookie({'name': 'PHPSESSID',
		 'value' : session,
		 'path' : '/'})
		
		browser.get(url)
	
		while 1:
			try:
				browser.switch_to_alert().accept()
	
			except selenium.common.exceptions.NoAlertPresentException:
				break
	
		print browser.title
		print time.strftime("%Y-%m-%d %X", time.localtime())
		time.sleep(2)
		browser.quit()
		time.sleep(10)
```

这样就比较合适的解决了问题。

ps:改脚本的时候其实有一点儿问题，这里的phpsession其实可以复用，因为默认有效时间大概是3小时，可以把判断改为判断session失效后调用登陆获取新的session。

在2天48小时的时间内，我的bot只挂了大概5次左右，其中两次是不小心被我们的运维ban了，有两次是在发起请求的时候超时导致卡死退出，还有一次目测是有个选手发了大概20条刷新，导致webdirver直接卡死退出了。

虽然不能说是完善的xss题目守护解决方案，不过也算是解决了大部分的情况，希望有人能提出更好的办法吧
