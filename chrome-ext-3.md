---
title: 从0开始入门Chrome Ext安全（番外篇） -- Zoomeye Tools
date: 2020-02-03 12:06:42
tags: chrome_ext
---



---

在经历了两次对Chrome Ext安全的深入研究之后，这期我们先把Chrome插件安全的问题放下来，这期我们将一个关于Chrome Ext的番外篇 -- Zoomeye Tools.

这篇文章让我们换一个角度，从开发一个插件开始，如何去审视chrome不同层级之间的问题。

这里我们主要的目的是完成一个Zoomeye的辅助插件。

- [Zoomeye Tools下载链接](https://chrome.google.com/webstore/detail/zoomeye-tools/bdoaeiibkccgkbjbmmmoemghacnkbklj)

<!--more-->

# 核心与功能设计

在zoomeye Tools中，我们主要加入了一下针对zoomeye的辅助性功能，在设计zoomeye Tools之前，首先我们需要思考我们需要什么样的功能。

这里我们需要需要实现的是两个大功能，

1、首先需要完成一个简易版本的zoomeye界面，用于显示当前域对应ip的搜索结果。
2、我们会完成一些zoomeye的辅助小功能，比如说一键复制搜索结果的左右ip等...

这里我们分别研究这两个功能所需要的部分：

## zoomeye minitools

关于Zoomeye的一些辅助小功能，这里我们首先拿一个需求来举例子，我们需要一个能够复制zoomeye页面内所有ip的功能，能便于方便的写脚本或者复制出来使用。

在开始之前，我们首先得明确chrome插件中不同层级之间的权限体系和通信方式：

在第一篇文章中我曾着重讲过这部分内容。
- [从0开始入门Chrome Ext安全（一） -- 了解一个Chrome Ext](https://lorexxar.cn/2019/11/22/chrome-ext-1/#%E6%9D%83%E9%99%90%E4%BD%93%E7%B3%BB%E5%92%8Capi)

我们需要完成的这个功能，可以简单量化为下面的流程：
```
用户点击浏览器插件的功能
-->
浏览器插件读取当前Zoomeye页面的内容
-->
解析其中内容并提取其中的内容并按照格式写入剪切板中
```

当然这是人类的思维，结合chrome插件的权限体系和通信方式，我们需要把每一部分拆解为相应的解决方案。

- 用户点击浏览器插件的功能

当用户点击浏览器插件的图标时，将会展示popup.html中的功能，并执行页面中相应加的js代码。

- 浏览器插件读取当前Zoomeye页面的内容

由于popup script没有权限读取页面内容，所以这里我们必须通过`chrome.tabs.sendMessage`来沟通content script，通过content script来读取页面内容。

- 解析其中内容并提取其中的内容并按照格式写入剪切板中

在content script读取到页面内容之后，需要通过`sendResponse`反馈数据。

当popup收到数据之后，我们需要通过特殊的技巧把数据写入剪切板
```
 function copytext(text){
    var w = document.createElement('textarea');
    w.value = text;
    document.body.appendChild(w);
    w.select();


    document.execCommand('Copy');

    w.style.display = 'none';
    return;
}
```
这里我们是通过新建了textarea标签并选中其内容，然后触发copy指令来完成。

整体流程大致如下

![image.png-31.5kB][1]

## zoomeye preview

与minitools的功能不同，要完成zoomeye preview首先我们遇到的第一个问题是zoomeye本身的鉴权体系。

在Zoomeye的设计中，大部分的搜索结果都需要登录之后使用，而且其相应的多种请求api都是通过jwt来做验证。

![image.png-99kB][2]

而这个jwt token会在登陆期间内储存在浏览器的local storage中。

![image.png-107.4kB][3]

我们可以简单的把架构画成这个样子

![image.png-41.5kB][4]

在继续设计代码逻辑之前，我们首先必须确定逻辑流程，我们仍然把流程量化为下面的步骤：
```
用户点击Zoomeye tools插件
-->
插件检查数据之后确认未登录，返回需要登录
-->
用户点击按钮跳转登录界面登录
-->
插件获取凭证之后储存
-->
用户打开网站之后点击插件
-->
插件通过凭据以及请求的host来获取zoomeye数据
-->
将部分数据反馈到页面中
```

紧接着我们配合chrome插件体系的逻辑，把前面步骤转化为程序逻辑流程。

- 用户点击Zoomeye tools插件

插件将会加载popup.html页面并执行相应的js代码。

- 插件检查数据之后确认未登录，返回需要登录

插件将获取储存在`chrome.storage`的zoomeye token，然后请求`zoomeye.org/user`判断登录凭据是否有效。如果无效，则会在popup.html显示need login。并隐藏其他的div窗口。

- 用户点击按钮跳转登录界面登录

当用户点击按钮之后，浏览器会直接打开
`https://sso.telnet404.com/cas/login?service=https%3A%2F%2Fwww.zoomeye.org%2Flogin`

如果浏览器当前在登录状态时，则会跳转回zoomeye并将相应的数据写到localStorage里。

- 插件获取凭证之后储存

由于前后端的操作分离，所有bg script需要一个明显的标志来提示需要获取浏览器前端的登录凭证，我把这个标识为定为了**当tab变化时，域属于zoomeye.org且未登录时**，这时候bg script会使用`chrome.tabs.executeScript`来使前端完成获取localStorage并储存进chrome.storage.

这样一来，插件就拿到了最关键的jwt token

- 用户打开网站之后点击插件

在完成了登录问题之后，用户就可以正常使用proview功能了。

当用户打开网站之后，为了减少数据加载的等待时间，bg script会直接开始获取数据。

- 插件通过凭据以及请求的host来获取zoomeye数据

后端bg script 通过判断tab状态变化，来启发获取数据的事件，插件会通过前面获得的账号凭据来请求

`https://www.zoomeye.org/searchDetail?type=host&title=`
并解析json，来获取相应的ip数据。

- 将部分数据反馈到页面中

当用户点击插件时，popup script会检查当前tab的url和后端全局变量中的数据是否一致，然后通过
```
bg = chrome.extension.getBackgroundPage();
```
来获取到bg的全局变量。然后将数据写入页面中。

整个流程的架构如下：

![image.png-52.9kB][5]

# 完成插件

在完成架构设计之后，我们只要遵守好插件不同层级之间的各种权限体系，就可以完成基础的设计，配合我们的功能，我们生成的manifest.json如下

```
{
	"name": "Zoomeye Tools",
	"version": "0.1.0",
	"manifest_version": 2,
	"description": "Zoomeye Tools provides a variety of functions to assist the use of Zoomeye, including a proview host and many other functions",
	"icons": {
		"16": "img/16_16.png",
		"48": "img/48_48.png",
		"128": "img/128_128.png"
	},
	"background": {
		"scripts": ["/js/jquery-3.4.1.js", "js/background.js"]
	},
	"content_scripts": [
		{
			"matches": ["*://*.zoomeye.org/*"],
			"js": ["js/contentScript.js"],
			"run_at": "document_end"
		}
	 ],
	"content_security_policy": "script-src 'self' 'unsafe-eval'; object-src 'self';",
	"browser_action": {
		"default_icon": {
			"19": "img/19_19.png",
			"38": "img/38_38.png"
		},
		"default_title": "Zoomeye Tools",
		"default_popup": "html/popup.html"
	},
	"permissions": [
		"clipboardWrite",
		"tabs",
		"storage",
		"activeTab",
		"https://api.zoomeye.org/",
		"https://*.zoomeye.org/"
	]
}
```

# 上传插件到chrome store

在chrome的某一个版本之后，chrome就不再允许自签名的插件安装了，如果想要在chrome上安装，那就必须花费5美金注册为chrome插件开发者。

并且对于chrome来说，他有一套自己的安全体系，如果你得插件作用于多个域名下，那么他会在审核插件之前加入额外的审核，如果想要快速提交自己的插件，那么你就必须遵守chrome的规则。

你可以在chrome的开发者信息中心完成这些。

![image.png-119.7kB][6]

# Zoomeye Tools 使用全解

## 安装

chromium系的所有浏览器都可以直接下载

- [https://chrome.google.com/webstore/detail/zoomeye-tools/bdoaeiibkccgkbjbmmmoemghacnkbklj](https://chrome.google.com/webstore/detail/zoomeye-tools/bdoaeiibkccgkbjbmmmoemghacnkbklj)

初次安装完成时应该为
![image.png-24.4kB][7]

## 使用方法

由于Zoomeye Tools提供了两个功能，一个是Zoomeye辅助工具，一个是Zoomeye preview.

### zoomeye 辅助工具

首先第一个功能是配合Zoomeye的，只会在Zoomeye域下生效，这个功能不需要登录zoomeye。

当我们打开Zoomeye之后搜索任意banner，等待页面加载完成后，再点击右上角的插件图标，就能看到多出来的两条选项。

![image.png-291.2kB][8]


如果我们选择copy all ip with LF，那么剪切板就是
```
23.225.23.22:8883
23.225.23.19:8883
23.225.23.20:8883
149.11.28.76:10443
149.56.86.123:10443
149.56.86.125:10443
149.233.171.202:10443
149.11.28.75:10443
149.202.168.81:10443
149.56.86.116:10443
149.129.113.51:10443
149.129.104.246:10443
149.11.28.74:10443
149.210.159.238:10443
149.56.86.113:10443
149.56.86.114:10443
149.56.86.122:10443
149.100.174.228:10443
149.62.147.11:10443
149.11.130.74:10443
```

如果我们选择copy all url with port
```
'23.225.23.22:8883','23.225.23.19:8883','23.225.23.20:8883','149.11.28.76:10443','149.56.86.123:10443','149.56.86.125:10443','149.233.171.202:10443','149.11.28.75:10443','149.202.168.81:10443','149.56.86.116:10443','149.129.113.51:10443','149.129.104.246:10443','149.11.28.74:10443','149.210.159.238:10443','149.56.86.113:10443','149.56.86.114:10443','149.56.86.122:10443','149.100.174.228:10443','149.62.147.11:10443','149.11.130.74:10443'
```

### Zoomeye Preview

第二个功能是一个简易版本的Zoomeye，这个功能需要登录Zoomeye。

在任意域我们点击右上角的Login Zoomeye，如果你之前登陆过Zoomeye那么会直接自动登录，如果没有登录，则需要在telnet404页面登录。登录完成后等待一会儿就可以加载完成。

在访问网页时，点击右上角的插件图标，我们就能看到相关ip的信息以及开放端口
![image.png-276.9kB][9]

# 写在最后

最后我们上传chrome开发者中心之后只要等待审核通过就可以发布出去了。

最终chrome插件下载链接：
- [Zoomeye Tools下载链接](https://chrome.google.com/webstore/detail/zoomeye-tools/bdoaeiibkccgkbjbmmmoemghacnkbklj)


[1]: http://static.zybuluo.com/LoRexxar/zlzcbjzsktu3w7bzrc39bbwt/image.png
[2]: http://static.zybuluo.com/LoRexxar/f7hdw9wo78ge7vsrguf85j4a/image.png
[3]: http://static.zybuluo.com/LoRexxar/qtabc0opzlamj5ine2i2359f/image.png
[4]: http://static.zybuluo.com/LoRexxar/r5r5hn1k3m0iyvc582bd79p8/image.png
[5]: http://static.zybuluo.com/LoRexxar/bk36oiqgv9k2l95hdkebvs7y/image.png
[6]: http://static.zybuluo.com/LoRexxar/t2v2fbuacthi7yxvne81k67i/image.png
[7]: http://static.zybuluo.com/LoRexxar/9nhywpk4wun6d8uf7tqdfeaj/image.png
[8]: http://static.zybuluo.com/LoRexxar/ktr8rus2dyswkcajdew7pg4q/image.png
[9]: http://static.zybuluo.com/LoRexxar/gzpp3gloo6yyfpmwq9piw6w2/image.png